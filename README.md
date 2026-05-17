### e27 Echelon Singapore · AI Workflow Competition 2026 - Boldr Customer Customer Intelligence Engine

AI-powered Customer Intelligence Engine for BOLDR — auto-drafts replies, detects knowledge gaps, benchmarks community sentiment, and generates marketing insights. Built with n8n, Qwen AI, Supabase &amp; Tavily. Submission for e27 Echelon Singapore AI Workflow Competition 2026 by Morning Wu


**Builder:** Morning Wu · [LinkedIn](https://www.linkedin.com/in/morningwu/) · morning@afterworkstartup.com

---

## What This Is

An end-to-end AI customer support system built for **Boldr**, a Singapore watch micro-brand. It automatically classifies incoming customer tickets, drafts replies from a live knowledge base, detects content gaps, benchmarks those gaps against real community sentiment, and generates periodic marketing briefs — all without human intervention in the loop.

**Live dashboard:** open `dashboard/index.html` in any browser (no server needed).

---

## System Architecture

```
Customer Ticket (CSV / email)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  WF1 · Core Intelligence Loop  (n8n)                     │
│                                                          │
│  Classify ticket → Search KB → Enough confidence?        │
│     YES → Draft reply with Qwen AI → Save                │
│     NO  → Log knowledge gap                              │
│              └→ Extract gap labels with Qwen             │
│              └→ Search Reddit / WatchUSeek / Amazon      │
│                 (Tavily) → Save to gap_sentiment         │
└──────────────────────────────────────────────────────────┘
        │
        ├── Knowledge Gap detected
        │         │
        │         ▼
        │   ┌─────────────────────────────────┐
        │   │  WF2 · Self-Improving KB Loop   │
        │   │  Dashboard resolves gap →        │
        │   │  Qwen drafts KB answer →         │
        │   │  Pending approval → Add to KB    │
        │   └─────────────────────────────────┘
        │
        └── Every 6 hours (5+ open gaps exist)
                  │
                  ▼
          ┌─────────────────────────────────────┐
          │  WF3 · Marketing Insight Generator  │
          │  Aggregate gaps → Qwen analysis →   │
          │  Score opportunities → Save brief   │
          └─────────────────────────────────────┘
```

---

## Prerequisites — API Keys & Accounts

You need **four** services. All have free tiers sufficient to run this project.

### 1. Supabase (database)
1. Create a free account at [supabase.com](https://supabase.com)
2. Create a new project
3. Go to **Project Settings → API** and copy:
   - **Project URL** → looks like `https://xxxx.supabase.co`
   - **Service Role Key** → starts with `eyJ...`

> Use the **service role key**, not the anon key — it bypasses Row Level Security so the workflows can write freely.

### 2. Qwen AI / DashScope (LLM)
1. Create an account at [dashscope-intl.aliyuncs.com](https://dashscope-intl.aliyuncs.com)
2. Go to **API Keys** and generate a new key → starts with `sk-...`

> Model used: `qwen-plus`
> Endpoint: `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions`

### 3. Tavily (real-time web search)
1. Create an account at [tavily.com](https://tavily.com)
2. Copy your **API Key** from the dashboard → starts with `tvly-...`

> Used by WF1 to search Reddit r/MicrobrandWatches, WatchUSeek, and Amazon for community sentiment on each detected knowledge gap.

### 4. n8n (workflow engine)

**Option A — Docker** (recommended):
```bash
cd n8n
docker compose up -d
# Open http://localhost:5678
```

**Option B — n8n Cloud**: [app.n8n.cloud](https://app.n8n.cloud) (free tier available)

---

## Setup Instructions

### Step 1 — Create the Database Tables (Supabase)

1. Open your Supabase project → **SQL Editor**
2. Copy the full contents of [`supabase/schema.sql`](supabase/schema.sql) and run it

This creates all 7 tables:

| Table | Purpose |
|---|---|
| `knowledge_base` | KB entries WF1 searches to draft replies |
| `tickets_processed` | One row per ticket processed by WF1 |
| `knowledge_gaps` | Questions the KB could not answer |
| `kb_additions` | Candidate KB entries pending human approval |
| `marketing_briefs` | Periodic AI-generated content strategy briefs (WF3) |
| `marketing_insights` | Per-gap marketing opportunity scores (WF3) |
| `gap_sentiment` | Community sentiment per gap label — Reddit, WatchUSeek, Amazon (WF1 inline) |

### Step 2 — Seed the Knowledge Base

WF1 searches `knowledge_base` to draft replies. Populate it from the source documents in `data/Knowledge Sources/`:

| File | Contents |
|---|---|
| `02_product_reference.txt` | Product specs, materials, pricing |
| `03a_rate_card_engraving.csv` | Engraving service prices |
| `03b_rate_card_servicing.csv` | Watch servicing prices |
| `04_faq_document.txt` | Customer FAQ |
| `05_SOP.txt` | Customer service SOPs |

Insert rows into `knowledge_base` via Supabase SQL Editor:
```sql
insert into knowledge_base (question, answer, theme, source) values
  ('What is the water resistance rating of the Expedition?',
   'The Boldr Expedition is rated 100m (10ATM). Safe for swimming and snorkelling, not diving.',
   'product_specs', 'product_reference'),
  ('Do you offer watch servicing for older models?',
   'Yes. Battery replacement SGD 35, Regulation Service SGD 85, Full Service from SGD 160.',
   'servicing', 'rate_card_servicing');
-- Add one row per key fact from each source document
```

### Step 3 — Import Workflows into n8n

1. Open n8n → **Workflows → Import from file**
2. Import in this order (WF2 and WF3 must be active before WF1 runs):

| File | Workflow | Trigger |
|---|---|---|
| `WF2_Self_Improving_KB_Loop.json` | Self-improving KB loop | Webhook — called from dashboard Resolve button |
| `WF3_Marketing_Insight_Generator.json` | Marketing insight generator | Schedule — every 6 hours when 5+ open gaps exist |
| `WF1_Core_Intelligence_Loop.json` | Core intelligence loop | Webhook — one call per ticket |

### Step 4 — Paste API Keys into Each Workflow

Each workflow has a **Configuration** node (the first Set node after the trigger). Open it and replace the placeholder values:

| Field | Value |
|---|---|
| `QWEN_API_KEY` | Your DashScope API key (`sk-...`) |
| `QWEN_MODEL` | `qwen-plus` *(leave as-is)* |
| `QWEN_BASE_URL` | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` *(leave as-is)* |
| `SUPABASE_URL` | Your Supabase project URL |
| `SUPABASE_KEY` | Your Supabase service role key |

**WF1 only** — also update the Tavily key inside the **Search Tavily Sources** code node:
```javascript
const TAVILY_KEY = 'tvly-YOUR_KEY_HERE';
```

### Step 5 — Activate Workflows

1. Activate **WF2** and **WF3** first
2. Activate **WF1** last
3. Copy the **WF1 webhook URL** from the Webhook trigger node — you'll use it to submit tickets

### Step 6 — Configure the Dashboard

Open `dashboard/index.html` in a text editor and update the two config constants near the top of the `<script>` section:

```javascript
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-anon-key';  // anon key is fine — dashboard is read-only
```

Then open `dashboard/index.html` directly in any browser. No server needed.

### Step 7 — Run the Demo

The sample tickets are in `data/01_customer_tickets.csv`. Send them to WF1 via the webhook:

```bash
curl -X POST http://localhost:5678/webhook/boldr-cs-intelligence \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": "TKT-1001",
    "subject": "Water resistance question",
    "customer_name": "Jane Tan",
    "customer_email": "jane@example.com",
    "customer_question": "Is the Expedition safe to wear while swimming?",
    "channel": "email",
    "date_received": "2026-01-15T09:00:00Z"
  }'
```

The dashboard auto-refreshes. Switch to each tab to see results populate in real time.

---

## Dashboard Tabs

| Tab | What it shows |
|---|---|
| **Overview** | KPI summary — tickets processed, auto-draft rate, gap rate |
| **Draft Replies** | AI-drafted replies ready to send; click any row to see the original question |
| **Knowledge Gaps** | Unanswered questions; click Resolve to trigger WF2 |
| **KB Additions** | Pending and approved KB entries generated by WF2 |
| **Weekly Report** | Theme, persona, outcome, and channel breakdowns |
| **Marketing Intel Brief** | Qwen-generated content strategy brief aggregated from gap patterns |
| **Sentiment Benchmark** | Community sentiment per gap label — sourced from Reddit, WatchUSeek, Amazon |
| **Builder** | About the project and the builder |

---

## Project Structure

```
e27 Builder Challenge/
├── README.md
├── dashboard/
│   └── index.html                   # Single-file dashboard — open in browser
├── data/
│   ├── 01_customer_tickets.csv      # Sample tickets for the demo
│   └── Knowledge Sources/           # Source documents to seed knowledge_base
│       ├── 02_product_reference.txt
│       ├── 03a_rate_card_engraving.csv
│       ├── 03b_rate_card_servicing.csv
│       ├── 04_faq_document.txt
│       └── 05_SOP.txt
├── n8n/
│   ├── docker-compose.yml           # Run n8n locally with Docker
│   ├── .env.example                 # Environment variable reference
│   └── workflows/
│       ├── WF1_Core_Intelligence_Loop.json
│       ├── WF2_Self_Improving_KB_Loop.json
│       └── WF3_Marketing_Insight_Generator.json
└── supabase/
    └── schema.sql                   # Run once in Supabase SQL Editor
```

---

## Resetting the Database (Testing)

To wipe all data and start fresh, run in Supabase SQL Editor:

```sql
truncate table tickets_processed   restart identity cascade;
truncate table knowledge_gaps      restart identity cascade;
truncate table kb_additions        restart identity cascade;
truncate table marketing_briefs    restart identity cascade;
truncate table marketing_insights  restart identity cascade;
truncate table gap_sentiment       restart identity cascade;
```

> `knowledge_base` is intentionally excluded — you don't want to lose your KB seed data.

---

## Built With

| Tool | Role |
|---|---|
| **n8n** | Workflow automation engine |
| **Qwen AI (qwen-plus)** | LLM for ticket classification, KB search, reply drafting, and insight generation |
| **Supabase** | PostgreSQL database + instant REST API |
| **Tavily** | Real-time web search for community sentiment |
| **Vanilla HTML/JS** | Dashboard — zero dependencies, opens directly in browser |

---

*Built by [Morning Wu](https://www.linkedin.com/in/morningwu/) for the e27 Echelon Singapore AI Workflow Competition 2026.*
*Founder of [AfterWork Startup](https://app.afterworkstartup.com) — helping 1M non-techies master practical AI skills.*
