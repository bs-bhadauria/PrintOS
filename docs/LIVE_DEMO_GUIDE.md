# 🎮 PrintOS — Interactive Live Demo Guide

Visitors and reviewers can explore the live deployed web application directly from their phone or computer browser without needing a physical printer connected!

---

## 🔗 Step 1: Open the Live Application

Visit the production web portal hosted on Vercel:

👉 **[https://print-os-customer-web.vercel.app/?shop=bhadauria-prints](https://print-os-customer-web.vercel.app/?shop=bhadauria-prints)**

*(Or visit the base URL at [https://print-os-customer-web.vercel.app/](https://print-os-customer-web.vercel.app/))*

---

## 🧪 Step 2: Test Core Interactive Capabilities

### Test 1: Upload a PDF Document
1. Drag and drop any `.pdf` document or click the upload area to select one from your device.
2. Observe:
   - Instant page count calculation.
   - Immediate addition to your shopping cart.
   - Dynamic rate calculation based on active shop tariffs.

### Test 2: Test Image-to-A4 Normalization
1. Select any mobile photo or screenshot (`.png`, `.jpg`, or `.jpeg`).
2. Upload the image.
3. Observe:
   - The system automatically accepts the image format.
   - On the backend, `pdf-lib` encapsulates and centers the image into a standardized A4 page with clean `20pt` margins.
   - The file is added with page count `1`.

### Test 3: Test Dynamic Page Range Slicing
1. Upload a multi-page PDF (e.g., 5 or 10 pages).
2. Inside the document configuration card, locate the **Page Range** input box.
3. Type: `1, 3`
4. Observe:
   - The estimated page count immediately drops to **2 pages**.
   - The total price automatically updates in real-time.

### Test 4: Checkout & Live Tracking
1. Click **Proceed to Checkout**.
2. Select **Pay by Cash on Counter** (or simulated instant payment).
3. Confirm the order.
4. Watch the 3-Stage Live Status Tracker:
   - Stage 1: **Payment Verified** (Instantly lights up green).
   - Real-time tracking code is displayed.
   - WebSocket connection status shows active streaming.

---

## 🏪 Step 3: Explore the Merchant Dashboard

The merchant dashboard demonstrates how shopkeepers view their live print queue in real time:

👉 **[https://print-os-customer-web.vercel.app/dashboard](https://print-os-customer-web.vercel.app/dashboard)**

* **Live Queue**: Real-time display of arriving print jobs.
* **Pricing Settings**: Configuration of Color, B&W, and Duplex page rates.
* **Cash Verification Controls**: Instant trigger for counter payment confirmations.
