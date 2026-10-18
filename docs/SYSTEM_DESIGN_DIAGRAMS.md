# 📊 PrintOS — System Design & Flow Diagrams

This document contains detailed visual representations of the PrintOS architecture, sequence flows, and edge execution models.

---

## 1. End-to-End System Sequence Flow

```mermaid
sequenceDiagram
    autonumber
    actor Customer as 📱 Customer Phone
    participant Web as ⚡ Customer Web (Vercel)
    participant API as 🚀 Orchestrator API (Render)
    participant Engine as ⚙️ Document Engine (pdf-lib)
    participant DB as 🐘 Neon PostgreSQL
    participant Socket as 📡 WebSocket Server
    participant Agent as 🖥️ Desktop Agent (Windows)
    participant Spooler as 🖨️ Hardware Spooler

    Customer->>Web: 1. Scan Shop QR & Open App
    Web->>API: 2. GET /api/shops/:shopId
    API->>DB: 3. Query Shop Rates & Config
    DB-->>API: 4. Return Data
    API-->>Web: 5. Render Shop UI & Tariffs

    Customer->>Web: 6. Upload Image / PDF
    Web->>API: 7. POST /api/upload (Multipart)
    alt Upload is Image (.png/.jpg)
        API->>Engine: Convert Image to Centered A4 PDF
        Engine-->>API: Standardized PDF
    end
    API-->>Web: 8. Return { filename, pages: N }

    Customer->>Web: 9. Configure Sidedness, Copies, Range (e.g. 1, 3)
    Note over Web: Instant Dynamic Price Estimate
    Customer->>Web: 10. Confirm Checkout & Payment
    Web->>API: 11. POST /api/jobs
    alt Custom Page Range Specified
        API->>Engine: Slice PDF to Selected Pages (1, 3)
        Engine-->>API: sliced-uuid.pdf
    end
    API->>DB: 12. Create PrintJob (Status: PAID)
    API->>Socket: 13. Emit 'new-print-job' to Shop Agent

    Socket->>Agent: 14. Deliver Print Payload
    Agent->>API: 15. Download Document (GET /api/download/:filename)
    API-->>Agent: 16. Stream Sliced PDF
    Agent->>Spooler: 17. Headless Spool Command (SumatraPDF)
    Spooler-->>Customer: 18. Physical Pages Printed!

    Agent->>Socket: 19. Emit 'job-completed' Handshake
    Socket->>DB: 20. Update PrintJob (Status: COMPLETED)
    Socket->>Web: 21. Push Live Status: READY_TO_COLLECT
    Note over API,Agent: 22. Automatic Ephemeral Storage Purge
```

---

## 2. Print Job State Machine

```mermaid
stateDiagram-v2
    [*] --> PENDING: Job Created (Customer Uploads)
    PENDING --> PAID: Payment Verified (Online / UPI)
    PENDING --> CASH_AWAITING: Pay by Cash Selected
    CASH_AWAITING --> PAID: Shopkeeper Confirms Cash on Counter
    PAID --> QUEUED: Agent Offline / Buffered in DB
    PAID --> PRINTING: Agent Connected & Spool Dispatched
    QUEUED --> PRINTING: Agent Reconnects & Syncs Queue
    PRINTING --> COMPLETED: Native Spooler Handshake Confirmed
    COMPLETED --> PURGED: Ephemeral Storage Cleaned Up
    PURGED --> [*]
```

---

## 3. Distributed Network Topology

```mermaid
graph LR
    subgraph WAN ["Public Cloud Network (HTTPS / WSS)"]
        A["📱 Mobile Devices"] -->|HTTPS| B["⚡ Vercel Edge CDN"]
        B -->|Reverse Proxy| C["🚀 Render Web Service (Node.js)"]
        C <-->|TCP TLS Pooled| D["🐘 Neon Serverless PostgreSQL"]
    end

    subgraph LAN ["Shop Local Network (Behind NAT / Firewall)"]
        C <-->|Outbound WSS TLS Tunnel| E["🖥️ PrintOS Desktop Agent"]
        E -->|Win32 API Spool| F["🖨️ Windows Print Spooler"]
        F -->|USB / Wi-Fi / Ethernet| G["📠 Commercial Printer"]
    end
```

---

## 4. Multi-Tenant Entity Relationship Model (ERD)

```mermaid
erDiagram
    SHOP ||--o{ PRINT_JOB : receives
    SHOP {
        string id PK
        string name
        string secretKey
        string passwordHash
        float bwRate
        float colorRate
        float bwDuplexRate
        float colorDuplexRate
        datetime createdAt
    }

    PRINT_JOB {
        string id PK
        string shopId FK
        string status
        json files
        float totalPrice
        float bwRateApplied
        float colorRateApplied
        datetime createdAt
        datetime updatedAt
    }
```
