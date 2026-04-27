# n8n-iphone-rental-automation-workflow
# 📱 End-to-End iPhone Rental Operations Automation

> **Note:** The raw n8n workflow JSON files are proprietary and not publicly available. This repository serves as an engineering case study demonstrating the architecture, design decisions, and business impact of the system.

---

## 📌 Executive Summary

Managing an iPhone rental business involves juggling customer inquiries, real-time inventory checks, payment tracking, and manual stock returns — all of which are prone to human error when handled manually.

I designed and deployed a **fully automated, end-to-end operations system** using n8n to handle the entire rental lifecycle — from the first customer message on WhatsApp to automatic order cancellation for unpaid transactions — with zero manual intervention for the core operational flow.

This project was built as a hands-on automation practice case, targeting the operational needs of **UMKM (small businesses) and freelancers** who manage rentals without a dedicated admin team.

---

## ⚠️ The Problem

Before automation, a typical iPhone rental operation faces these recurring bottlenecks:

1. **Manual Availability Checks** — Staff must cross-reference inventory spreadsheets every time a customer asks about a unit.
2. **Inconsistent Order Tracking** — Orders received via chat are easily lost, creating gaps between inquiry and fulfillment.
3. **Slow & Error-Prone Invoicing** — Manually generating payment links and updating payment status wastes time and causes delays.
4. **No Auto-Cancellation** — Unpaid orders stay open indefinitely, blocking inventory that could go to other customers.
5. **Inventory Sync Lag** — Stock updates from walk-in returns don't propagate to the system in real time.

---

## 💡 The Solution

I built a **modular, event-driven architecture** that connects a WhatsApp chatbot, an order intake form, a payment gateway, a relational database, and a spreadsheet — all orchestrated by n8n — without requiring manual touchpoints for standard transactions.

### High-Level Flow

```
Customer Message (WhatsApp)
    ↓
Subflow A: Chatbot — greets customer, checks iPhone availability in real time
    ↓ (customer proceeds to order)
Subflow B: Order Intake — customer submits form (Tally.so) → 
           n8n processes data → creates record in PostgreSQL & Google Sheets →
           generates iPaymu payment link → sends to customer via WhatsApp
    ↓
               ┌──────────────────────────────────┐
               │                                  │
         [Payment Received]               [No Payment in 24h]
               │                                  │
        Subflow C:                         Subflow D:
  iPaymu Webhook → update                 Auto-cancel order →
  PostgreSQL & Google Sheets              update status to CANCELED
  to PAID status                          in PostgreSQL & Google Sheets
               │
        Subflow E (Admin):
  Admin updates Google Sheet for
  stock returns → auto-synced to PostgreSQL
```

---

## 🏗️ Architecture: 5 Subflows

The system is broken into **5 independent subflows**, each responsible for a single business domain. This decoupled design ensures each subflow can be modified or debugged without affecting the others.

| Subflow | Name | Trigger | Responsibility |
|---|---|---|---|
| **A** | WhatsApp Chatbot | Incoming WhatsApp message | Greet customer, check & display iPhone availability |
| **B** | Order Processing | Tally.so form submission (Webhook) | Validate order, insert to DB, generate & send iPaymu payment link |
| **C** | Payment Confirmation | iPaymu payment webhook | Update order status to `PAID` in PostgreSQL & Google Sheets |
| **D** | Auto-Cancellation | n8n Cron (scheduled) | Cancel unpaid orders older than 24 hours, update status to `CANCELED` |
| **E** | Inventory Sync | Google Sheets change | Sync admin stock updates (e.g., walk-in returns) from Sheet to PostgreSQL |

---

## ✨ Engineering Highlights

### Real-Time Availability Check
The WhatsApp chatbot queries PostgreSQL directly to return live inventory data. Customers always see accurate stock without any manual refresh.

### Idempotent Order Inserts
Order records are written with a unique constraint on the order identifier, preventing duplicate entries if a webhook fires more than once (common in payment gateway integrations).

### Dual-Write Strategy
All status updates (PAID, CANCELED) are written to **both** PostgreSQL (source of truth) and **Google Sheets** (admin-facing dashboard) in the same workflow execution, keeping both layers consistently in sync.

### Scheduled Auto-Cancellation
A cron-triggered subflow queries the database for orders with status `PENDING` older than 24 hours and bulk-updates them to `CANCELED`, freeing up inventory automatically.

### Intentional Design Decision — Manual Walk-In Stock Returns
Admin manually updates the Google Sheet when a unit is returned in person (walk-in). This was a deliberate choice: at UMKM scale, a lightweight spreadsheet update is simpler and more reliable than an additional confirmation layer. Subflow E auto-syncs any Sheet changes to PostgreSQL within minutes.

> **Future Enhancement Considered:** A Telegram bot confirmation flow for walk-in returns (admin sends a command → bot updates DB directly), eliminating the Sheet dependency for this edge case.

---

## 🛠️ Tech Stack

| Layer | Tool |
|---|---|
| **Automation Orchestrator** | [n8n](https://n8n.io) (self-hosted) |
| **Customer Channel** | WhatsApp Business API |
| **Order Intake** | [Tally.so](https://tally.so) (webhook form) |
| **Payment Gateway** | [iPaymu](https://ipaymu.com) |
| **Database** | PostgreSQL |
| **Admin Dashboard / Inventory** | Google Sheets |
| **Documentation** | Notion |

---

## 🗄️ Database Schema (Simplified)

```sql
-- Core orders table
CREATE TABLE orders (
    id              SERIAL PRIMARY KEY,
    order_id        VARCHAR(50) UNIQUE NOT NULL,
    customer_name   VARCHAR(100),
    customer_phone  VARCHAR(20),
    iphone_model    VARCHAR(50),
    rental_start    DATE,
    rental_end      DATE,
    total_price     NUMERIC(12, 2),
    status          VARCHAR(20) DEFAULT 'PENDING',
    payment_url     TEXT,
    paid_at         TIMESTAMPTZ,
    canceled_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Inventory table (synced from Google Sheets via Subflow E)
CREATE TABLE inventory (
    id                  SERIAL PRIMARY KEY,
    iphone_model        VARCHAR(50) UNIQUE NOT NULL,
    stock_available     INT DEFAULT 0,
    synced_from_sheet_at TIMESTAMPTZ
);
```

---

## 📖 What I Learned

- **Webhook reliability**: Payment gateways can fire duplicate webhooks. Designing idempotent handlers is non-negotiable.
- **Dual-write tradeoffs**: Keeping PostgreSQL and Google Sheets in sync adds complexity but gives non-technical admins a familiar interface with zero training overhead.
- **Subflow modularity pays off**: Isolating each business domain into its own subflow made debugging significantly faster — a bug in payment confirmation never touched the chatbot logic.
- **Designing for the actual user**: At UMKM scale, "good enough + simple" often outperforms "perfect + complex". The manual walk-in return process is a direct result of this principle.

---

## 📬 Contact

Interested in a similar automation system for your business operations?  
Feel free to reach out — I'm open to discussing custom workflow architecture.

---

*Built with n8n · PostgreSQL · Google Sheets · WhatsApp · iPaymu · Tally.so*
