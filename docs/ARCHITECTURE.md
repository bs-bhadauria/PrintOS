# 🏛️ PrintOS — System Architecture & Engineering Deep-Dive

## 1. System Overview

PrintOS is designed to operate seamlessly across high-latency, intermittently-connected local retail environments while maintaining a centralized, low-latency cloud control plane.

The architecture solves three core challenges in retail printing automation:
1. **Network NAT / Firewall Traversal**: Local shop computers are behind residential/retail NAT routers without public IP addresses or port forwarding capabilities.
2. **Printer Driver Fragmentation**: Print shops utilize diverse printer manufacturers (HP, Canon, Epson, Brother, Ricoh) across USB, local LAN, and Wi-Fi connections.
3. **High File Ingestion Variance**: Customers upload documents in various file formats (multi-page PDFs, mobile camera photos, screenshots) with diverse dimensions.

---

## 2. Distributed Component Topology

```
+-----------------------------------------------------------------------------------+
|                              CLOUD CONTROL PLANE                                  |
|                                                                                   |
|   +-----------------------+                    +------------------------------+   |
|   |  Customer Web Portal  | <--(HTTPS / CDN)-- | Vercel Global Edge Network   |   |
|   |  (Next.js 14 App Rtr) |                    | (PWA, SSR, Static Assets)    |   |
|   +-----------------------+                    +------------------------------+   |
|              |                                                                    |
|           (HTTPS)                                                                 |
|              v                                                                    |
|   +-----------------------+                    +------------------------------+   |
|   |   Node.js / Express   | <----------------> |   Neon Serverless Postgres   |   |
|   |   API Orchestrator    |                    |   (Prisma ORM, Pooled SQL)   |   |
|   |   (Hosted on Render)  |                    +------------------------------+   |
|   +-----------------------+                                                       |
|              |                                                                    |
|       (Event Pipeline)                                                            |
|              v                                                                    |
|   +-----------------------+                                                       |
|   |  Socket.IO Real-Time  |                                                       |
|   |  WebSocket Server     |                                                       |
|   +-----------------------+                                                       |
+--------------|--------------------------------------------------------------------+
               |
         (WSS TLS Tunnel)
               |
+--------------v--------------------------------------------------------------------+
|                         EDGE EXECUTION PLANE (ON-PREMISES)                        |
|                                                                                   |
|   +---------------------------------------------------------------------------+   |
|   |  Shop Windows Desktop (Windows 10 / 11)                                    |   |
|   |                                                                           |   |
|   |   +-----------------------+                 +-------------------------+   |   |
|   |   |  printos-agent.vbs    | -(Runs Hidden)->|   print-agent.exe       |   |   |   |
|   |   |  (Startup Daemon)     |                 |   (PKG Standalone Node) |   |   |   |
|   |   +-----------------------+                 +-------------------------+   |   |
|   |                                                          |                |   |
|   |                                                    (CLI Spool)            |   |
|   |                                                          v                |   |
|   |                                             +-------------------------+   |   |
|   |                                             |  SumatraPDF CLI Driver  |   |   |
|   |                                             +-------------------------+   |   |
|   |                                                          |                |   |
|   |                                                    (Win32 Spool)          |   |
|   |                                                          v                |   |
|   |                                             +-------------------------+   |   |
|   |                                             |  Windows Print Spooler  |   |   |
|   |                                             +-------------------------+   |   |
|   +----------------------------------------------------------|----------------+   |
|                                                              v                    |
|                                                 +-------------------------+       |
|                                                 | Physical Printer Device |       |
|                                                 | (USB / LAN / Wi-Fi)     |       |
|                                                 +-------------------------+       |
+-----------------------------------------------------------------------------------+
```

---

## 3. Communication Protocol & Resilient Edge Agent Design

### Outbound-Only Persistent WebSocket
Because retail shop computers sit behind commercial or mobile ISP NATs with dynamic IPs, traditional webhook or inbound REST architectures are impossible without complex tunneling. 

PrintOS utilizes an **outbound-initiated, long-lived WebSocket connection** (`Socket.IO` over TLS) from the shop's desktop agent to the central backend. Once established, the backend can push real-time print commands down to the agent instantly.

### Silent Auto-Boot Daemon (`printos-agent.vbs`)
Shopkeepers should not have to manually launch command prompts or leave open console windows. 
1. During the initial agent CLI wizard, the agent checks if auto-start is requested.
2. It generates a VBScript launcher in `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\printos-agent.vbs`.
3. The VBScript calls `WshShell.Run` with window style `0` (Completely Hidden):
   ```vbscript
   Set WshShell = CreateObject("WScript.Shell")
   WshShell.Run """C:\Path\To\print-agent.exe""", 0, False
   ```
4. All `stdout` and `stderr` streams are automatically routed to a persistent local `agent.log` file next to the binary.

### Robust Working Directory (`CWD`) Normalization
When Windows invokes startup scripts from the Startup folder, Windows sets the default working directory to `C:\Windows\System32` or the Startup directory. The PrintOS agent uses executable-relative path resolution (`path.dirname(process.execPath)`) to guarantee that `config.json`, `temp/`, and `agent.log` are always resolved relative to the actual application binary, avoiding orphaned files.

### Edge Queue Recovery (Network Partition Resilience)
If a shop loses internet connectivity or the desktop computer is turned off while online customers submit jobs:
1. Customers can still pay and complete orders; jobs are stored in the Neon database with status `'PAID'`.
2. When the local agent's network recovers and the WebSocket re-authenticates via the `register-shop` handshake, the backend queries:
   ```sql
   SELECT * FROM "PrintJob" 
   WHERE "shopId" = :shopId AND "status" = 'PAID' 
   ORDER BY "createdAt" ASC;
   ```
3. All pending paid jobs are sequentially spooled down to the edge agent, ensuring zero dropped print orders.

---

## 4. Document Processing & Ingestion Pipeline

### On-the-Fly Image-to-A4 Normalizer (`pdf-lib`)
When mobile users upload phone screenshots or photos (`.jpg`, `.jpeg`, `.png`), feeding raw images to native printer drivers often causes margin clipping, stretched aspect ratios, or orientation issues.

PrintOS intercepts raw images inside the `/api/upload` endpoint:
1. The server loads the raw image buffer using `PDFDocument` from `pdf-lib`.
2. A standard ISO 216 A4 page canvas is created:
   $$\text{Width} = 595.28 \text{ pt}, \quad \text{Height} = 841.89 \text{ pt}$$
3. The image is scaled proportionally to fit within the printable bounds while reserving a safe uniform `20pt` perimeter margin:
   $$\text{Scale} = \min\left(\frac{595.28 - 40}{\text{ImgWidth}}, \frac{841.89 - 40}{\text{ImgHeight}}\right)$$
4. The image is centered precisely along the horizontal and vertical axes:
   $$X = \frac{595.28 - \text{FinalWidth}}{2}, \quad Y = \frac{841.89 - \text{FinalHeight}}{2}$$
5. The original image is deleted from disk, and the request is transparently forwarded as a standardized 1-page A4 PDF.

### Dynamic PDF Page Range Slicing
When a user inputs a page range (e.g. `1, 3, 5-8`):
1. A range parsing algorithm evaluates boundaries against the document's total page count:
   $$R \subseteq [1, \text{TotalPages}]$$
2. The server loads the source PDF and invokes `dstDoc.copyPages(srcDoc, pageIndices)`.
3. An isolated sliced document (`sliced-<uuid>.pdf`) is generated and saved.
4. Billing rates are calculated exclusively against the cardinality of the slice ($|R|$) rather than total document length.

---

## 5. Native Print Spooling via SumatraPDF

The edge agent interfaces directly with the native Windows Print Spooler (`winspool.drv`) via a bundled, headless SumatraPDF binary:

```bash
SumatraPDF.exe -print-to "<PrinterName>" -print-settings "<Color>,<Duplex>,<PaperSize>" "<FilePath>"
```

### Advantages of Headless Driver Execution:
* **Zero User Prompts**: Bypasses all interactive Windows print dialogs.
* **Driver Universality**: Leverages standard PostScript / PCL / GDI printer drivers already configured on the shopkeeper's OS.
* **Non-Blocking Execution**: Asynchronous Node.js child processes monitor exit codes to confirm successful handoff to the hardware spooler before triggering the cloud completion handshake.
