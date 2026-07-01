# ShipBuddy — Company Profile

## Company Overview

**Company Name:** ShipBuddy Inc.  
**Founded:** 2019  
**Headquarters:** Austin, Texas, USA  
**Stage:** Series B ($18M raised)  
**Employees:** 74 (12 in Customer Success)  
**Website:** shipbuddy.io  
**Tagline:** *"Ship smarter, not harder."*

---

## Mission

ShipBuddy helps e-commerce merchants of all sizes stop losing money on shipping. We automate multi-warehouse inventory management, intelligent order routing, and carrier rate shopping — so merchants focus on selling, not logistics.

---

## Product Overview

ShipBuddy is a cloud-based e-commerce operations platform that connects to major sales channels (Shopify, WooCommerce, Amazon, eBay, Etsy) and shipping carriers (UPS, FedEx, USPS, DHL, ShipStation-compatible carriers) through a single dashboard.

### Core Modules

| Module | Description |
|---|---|
| **Inventory Hub** | Real-time multi-warehouse stock sync across all sales channels |
| **Order Router** | Rule-based and AI-assisted order routing to the optimal warehouse/carrier |
| **Rate Shopper** | Live carrier rate comparison at checkout and fulfillment time |
| **Returns Portal** | Branded self-service return label generation for customers |
| **Analytics Dashboard** | Shipping cost, delivery performance, and returns analytics |
| **Developer API** | REST API + Webhooks for custom integrations |

---

## Customer Segments

### Segment 1: Solo Sellers ("Starters")
- 1 warehouse / 1 sales channel
- 50–500 orders/month
- Pain: Manual label printing, no rate shopping
- Plan: **Starter** ($49/month)

### Segment 2: Growing Brands ("Growth")
- 1–3 warehouses / 2–5 sales channels
- 500–5,000 orders/month
- Pain: Overselling, split shipments, rising carrier costs
- Plan: **Growth** ($199/month)

### Segment 3: Mid-Market Operations ("Pro")
- 3–10 warehouses / 5+ sales channels
- 5,000–50,000 orders/month
- Pain: Routing complexity, 3PL coordination, compliance
- Plan: **Pro** ($599/month)

### Segment 4: Enterprise
- Custom warehouse networks, dedicated account manager
- 50,000+ orders/month
- Plan: **Enterprise** (custom pricing)

---

## Pricing Plans

| Feature | Starter | Growth | Pro | Enterprise |
|---|---|---|---|---|
| Price | $49/mo | $199/mo | $599/mo | Custom |
| Orders/month | 500 | 5,000 | 50,000 | Unlimited |
| Warehouses | 1 | 3 | 10 | Unlimited |
| Sales Channels | 1 | 5 | Unlimited | Unlimited |
| API Access | Read-only | Full | Full | Full |
| Returns Portal | - | Yes | Yes | Yes |
| Dedicated CSM | - | - | Yes | Yes |
| SLA | 99.5% | 99.5% | 99.9% | 99.99% |

**Billing:** Monthly or Annual (20% discount). Credit card or ACH.  
**Trial:** 14-day free trial, no credit card required.  
**Overage:** Orders beyond plan limit are billed at $0.02/order.

---

## Technology Stack (Internal Reference)

- **Backend:** Python (FastAPI), PostgreSQL, Redis, Kafka
- **Frontend:** Next.js, Tailwind CSS
- **Infrastructure:** AWS (EKS, RDS, S3, SQS)
- **Carrier Integrations:** EasyPost API
- **Sales Channel Integrations:** Native Shopify App, WooCommerce Plugin, Amazon SP-API, eBay API

---

## Support Structure

| Tier | Channel | Response SLA | Handled By |
|---|---|---|---|
| All plans | Email | 24 hours | Customer Success FTE (AI) |
| Growth+ | Live Chat | 4 hours | Customer Success FTE (AI) → Human escalation |
| Pro+ | Phone + Dedicated CSM | 1 hour | Human CSM |
| Enterprise | Dedicated Slack | 15 minutes | Human CSM |

**Escalation criteria:**
- Revenue impact > $500
- Data loss or security concern
- 3+ failed resolution attempts by AI
- Customer explicitly requests human
- Churn risk signals detected

---

## Key Contacts (Internal)

| Name | Role | Handles |
|---|---|---|
| Priya Nair | Head of Customer Success | Escalations, churn risk |
| Marcus Webb | Senior Engineer | Technical bugs, API issues |
| Sofia Reyes | Billing Manager | Billing disputes, refunds |
| Jake Turner | Onboarding Specialist | New customer setup |

**Escalation Email:** escalations@shipbuddy.io  
**Engineering Bugs:** bugs@shipbuddy.io

---

## Common Customer Pain Points (Known Issues as of Q1 2026)

1. Shopify webhook delays causing inventory sync lag (known issue, fix ETA: 2026-05-30)
2. DHL international rate quotes occasionally returning $0 (EasyPost upstream bug)
3. WooCommerce plugin v2.3.1 incompatibility with PHP 8.3 (patch v2.3.2 released 2026-04-15)
4. Amazon SP-API token refresh failure after 90-day credential rotation (documented workaround exists)
5. PDF label rendering broken on Safari 17.4 (CSS fix in progress)

---

## Company Values

- **Merchant-first:** Every decision starts with merchant impact
- **Radical transparency:** We tell customers what's broken before they ask
- **Speed with care:** Fast responses, accurate information — never guess
- **Escalate with dignity:** Handing off to a human is a success, not a failure
