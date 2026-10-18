# ✨ PrintOS — Feature Specification & Capabilities

## 1. Feature Matrix Overview

| Feature Category | Capability | Business / User Value |
|---|---|---|
| **Customer Access** | Instant Scan-to-Print QR Code | Zero app installation required; works on all iOS & Android devices. |
| **Media Ingestion** | On-the-Fly Image-to-A4 Normalizer | Upload mobile photos & screenshots; auto-centered with margins on A4. |
| **Document Control**| Dynamic PDF Page Range Slicing | Print only selected pages (`1, 3, 5-8`) with instant fair billing. |
| **Custom Settings** | Mixed Setting Cart Support | Configure different pages of the same document with distinct color/copy rules. |
| **Tracking** | 3-Stage Real-Time WebSocket Tracker | Live animated order feedback (`Payment Verified` ➔ `Printing` ➔ `Ready`). |
| **Payments** | Dual Payment Flow | Supports instant digital verification or traditional "Pay by Cash on Counter". |
| **Edge Print Daemon**| Silent Background Windows Service | Auto-boots on Windows startup with zero terminal popups; logs to file. |
| **Fault Tolerance** | Resilient Offline Queue Recovery | Unsent paid jobs automatically catch up when internet or PC reconnects. |
| **Privacy & Security**| Ephemeral Storage Auto-Purge | Zero document retention; files wiped from server and client after printing. |
| **Merchant Control** | Live Shop Dashboard | Real-time queue monitor, custom pricing tariffs, and revenue metrics. |

---

## 2. Customer Experience (Scan • Customize • Print)

### 1. Dynamic Shop Landing (`/print?shop=<shopId>`)
- Each partner shop receives a unique branded URL and customized QR code.
- Scanning the QR code instantly loads the shop's name, custom color/B&W tariffs, duplex discounts, and connection status.

### 2. Multi-Document Shopping Cart
- Customers can upload multiple files simultaneously in a single checkout session.
- Per-document configuration options:
  - **Color Mode**: Grayscale (B&W) or Full Color.
  - **Sidedness**: Single-sided or Duplex (Double-sided).
  - **Paper Size**: Standard A4 or Large A3.
  - **Copies**: 1 to 100+ copies with live multiplier.
  - **Page Range**: Selective page extraction.

### 3. Smart Image-to-PDF Conversion
- Handles `.png`, `.jpg`, and `.jpeg` formats effortlessly.
- Standardizes diverse mobile resolutions and orientations into uniform, professional A4 printouts.

### 4. Interactive Live Status Modal
- Once checkout is confirmed, the customer is presented with an intuitive tracking screen.
- Status updates stream in real-time over Socket.IO without requiring page refreshes.

---

## 3. Merchant / Shopkeeper Portal

### 1. Live Job Queue Monitor
- Visual queue displaying pending, active, and completed print jobs.
- Audio and visual alerts when a new job arrives.

### 2. Tariff Management
- Shop owners can independently configure their rates:
  - Single-sided B&W rate (e.g. ₹2.00)
  - Single-sided Color rate (e.g. ₹10.00)
  - Duplex B&W rate (e.g. ₹1.50 per page)
  - Duplex Color rate (e.g. ₹8.00 per page)
  - A3 multiplier support

### 3. Cash Verification Workflow
- For customers who select "Pay by Cash", the shopkeeper can verify receipt of physical cash on their dashboard, which immediately triggers the edge agent to spool the printout.

---

## 4. Edge Desktop Agent (`print-agent.exe`)

### 1. Self-Bootstrapping CLI Setup
- First-time launch presents an interactive setup wizard in the console:
  - Enter Shop Secret Key.
  - Enter Central Backend Server Target.
  - Select default printer.
  - Prompt: `Do you want PrintOS Agent to start automatically when Windows boots? [y/n]`.

### 2. Silent Daemon Mode
- When configured for auto-boot, Windows starts the application via `printos-agent.vbs`.
- Runs completely hidden from view—no taskbar clutter, no terminal windows.
- Automatically writes structured timestamps and logs to `agent.log`.

### 3. Automatic Spool Cleanup
- After SumatraPDF confirms handoff to the Windows print queue, the agent automatically deletes the temporary downloaded PDF from its local `temp/` folder.
