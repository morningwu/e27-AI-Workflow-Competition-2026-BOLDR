# Boldr CS Intelligence — AI Workflow System
### e27 Echelon Singapore · AI Workflow Competition 2026
**Builder:** Morning Wu · [LinkedIn](https://www.linkedin.com/in/morningwu/) · morning@afterworkstartup.com

---

## 🔗 Live Links

| | |
|---|---|
| 📊 **Live Dashboard** | https://boldercs.netlify.app/ |

---

## What This Is

An end-to-end AI customer support system built for **Boldr**, a Singapore watch micro-brand. It automatically classifies incoming customer tickets, drafts replies from a live knowledge base, detects content gaps, benchmarks those gaps against real community sentiment, and generates periodic marketing briefs — all without human intervention in the loop.

---

## System Architecture

```
Customer Ticket (submitted via dashboard or Supabase)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  WF1 · Core Intelligence Loop  (n8n Cloud)               │
│                                                          │
│  Load tickets from Supabase tickets_inbox                │
│  → Build KB from knowledge_sources table                 │
│  → Classify ticket → Search KB → Enough confidence?      │
│     YES → Draft reply with Qwen AI → Save                │
│     NO  → Log knowledge gap                              │
│              └→ Extract gap labels with Qwen             │
│              └→ Search Reddit / WatchUSeek / Amazon      │
│                 (Tavily) → Save to gap_sentiment         │
└──────────────────────────────────────────────────────────┘
        │
        ├── Knowledge Gap detected
        │         ▼
        │   ┌─────────────────────────────────┐
        │   │  WF2 · Self-Improving KB Loop   │
        │   │  Dashboard resolves gap →        │
        │   │  Qwen drafts KB answer →         │
        │   │  Pending approval → Add to KB    │
        │   └─────────────────────────────────┘
        │
        └── Every 6 hours (5+ open gaps exist)
                  ▼
          ┌─────────────────────────────────────┐
          │  WF3 · Marketing Insight Generator  │
          │  Aggregate gaps → Qwen analysis →   │
          │  Score opportunities → Save brief   │
          └─────────────────────────────────────┘
```

---

## Option A — Try the Live Demo (No Setup Needed)

1. Open → **https://boldercs.netlify.app/**
2. Go to the **🧪 Add Sample Data** tab
3. Click any quick-fill chip (e.g. "📦 Shipping time") to load a sample question
4. Click **Submit Ticket** — inserted into Supabase instantly
5. Go to [n8n Cloud](https://morningmaker.app.n8n.cloud) → open **WF1** → click **Execute Workflow**
6. Watch **Draft Replies** or **Knowledge Gaps** tab update with the AI result live

> Contact morning@afterworkstartup.com for n8n Cloud viewer access if needed.

---

## Option B — Self-Hosted Setup (Full Reproduction)

### Prerequisites — API Keys & Accounts

| Service | Purpose | Sign up |
|---|---|---|
| **Supabase** | Database + REST API | [supabase.com](https://supabase.com) — free |
| **Qwen AI / DashScope** | LLM for drafting + classification | [dashscope-intl.aliyuncs.com](https://dashscope-intl.aliyuncs.com) — free quota |
| **Tavily** | Real-time web search | [tavily.com](https://tavily.com) — free tier |
| **n8n** | Workflow automation | [app.n8n.cloud](https://app.n8n.cloud) — free tier |

---

### Step 1 — Create Database Tables (Supabase)

1. Create a new Supabase project → go to **SQL Editor**
2. Paste and run the full contents of [`supabase/schema.sql`](supabase/schema.sql)

Tables created:

| Table | Purpose |
|---|---|
| `knowledge_sources` | Raw KB source files (FAQ, product ref, rate cards) |
| `tickets_inbox` | Incoming tickets to be processed by WF1 |
| `tickets_processed` | Every ticket WF1 has processed |
| `knowledge_gaps` | Questions the KB could not answer |
| `kb_additions` | Candidate KB entries pending human approval |
| `marketing_briefs` | Periodic AI-generated content strategy briefs |
| `marketing_insights` | Per-gap marketing opportunity scores |
| `gap_sentiment` | Community sentiment per gap label |

---

### Step 2 — Seed the Data

#### Upload tickets
1. Supabase → **Table Editor** → `tickets_inbox` → **Import data from CSV**
2. Upload `data/01_customer_tickets.csv`

#### Upload knowledge sources
**Table Editor** → `knowledge_sources` → **Insert row** — 4 rows total:

| `name` field | File to open and paste into `content` field |
|---|---|
| `faq` | `data/Knowledge Sources/04_faq_document.txt` |
| `product_reference` | `data/Knowledge Sources/02_product_reference.txt` |
| `engraving_rates` | `data/Knowledge Sources/03a_rate_card_engraving.csv` |
| `servicing_rates` | `data/Knowledge Sources/03b_rate_card_servicing.csv` |

Open each file in a text editor → Select All → Copy → paste into the `content` field.

---

### Step 3 — Import Workflows into n8n

Sign up at [app.n8n.cloud](https://app.n8n.cloud) → **Workflows → Import from file**

Import in this order (WF2 and WF3 must be active before WF1 runs):

| File | Workflow | Trigger |
|---|---|---|
| `WF2_Self_Improving_KB_Loop.json` | Self-improving KB loop | Webhook |
| `WF3_Marketing_Insight_Generator.json` | Marketing insight generator | Schedule — every 6 hours |
| `WF1_Core_Intelligence_Loop.json` | Core intelligence loop | Manual |

---

### Step 4 — Paste API Keys

In every workflow, open the **Configuration** node and fill in:

| Field | Value |
|---|---|
| `QWEN_API_KEY` | Your DashScope key (`sk-...`) |
| `QWEN_MODEL` | `qwen-plus` *(leave as-is)* |
| `QWEN_BASE_URL` | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1/chat/completions` *(leave as-is)* |
| `SUPABASE_URL` | Your Supabase project URL |
| `SUPABASE_KEY` | Your Supabase **service role key** |

**WF1 only** — also update the Tavily key inside the **Search Tavily Sources** code node:
```javascript
const TAVILY_KEY = 'tvly-YOUR_KEY_HERE';
```

Also update the **Call WF3 Marketing Insight** node URL to your own n8n webhook URL.

---

### Step 5 — Configure the Dashboard

Open `dashboard/index.html` and update these lines near the top of the `<script>` section:

```javascript
const SUPABASE_URL  = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_ANON = 'your-anon-key';
const SUPABASE_SVC  = 'your-service-role-key';
const WF2_WEBHOOK   = 'https://YOUR-N8N/webhook/resolve-gap';
const WF3_WEBHOOK   = 'https://YOUR-N8N/webhook/marketing-insight';
```

Deploy to [Netlify Drop](https://netlify.com/drop) by dragging the `dashboard/` folder.

---

### Step 6 — Run

1. Activate **WF2** and **WF3** first, then **WF1**
2. n8n → **WF1** → click **Execute Workflow**
3. Dashboard updates automatically

> **Batch size:** set `TICKET_LIMIT` in WF1's Configuration node (default `10`). Set to `0` for all tickets.

---

### Reset Database

```sql
truncate table tickets_processed   restart identity cascade;
truncate table knowledge_gaps      restart identity cascade;
truncate table kb_additions        restart identity cascade;
truncate table marketing_briefs    restart identity cascade;
truncate table marketing_insights  restart identity cascade;
truncate table gap_sentiment       restart identity cascade;

-- Reset ticket processing flag so WF1 picks them up again
update tickets_inbox set processed = false;
```

> `knowledge_sources` and `tickets_inbox` data is preserved.

---

## Dashboard Tabs

| Tab | What it shows |
|---|---|
| **Overview** | KPI summary — tickets processed, auto-draft rate, gap rate |
| **Draft Replies** | AI-drafted replies ready to send; click row to see original question |
| **Knowledge Gaps** | Unanswered questions; click Resolve to trigger WF2 |
| **KB Additions** | Pending and approved KB entries generated by WF2 |
| **Weekly Report** | Theme, persona, outcome, and channel breakdowns |
| **Marketing Intel Brief** | Qwen-generated content strategy brief from gap patterns |
| **Sentiment Benchmark** | Community sentiment per gap label — Reddit, WatchUSeek, Amazon |
| **🧪 Add Sample Data** | Submit a new ticket live to see the full AI pipeline run |
| **👤 Builder** | About the project and builder |

---

## Project Structure

```
live deploy/
├── README.md
├── dashboard/
│   └── index.html                        # Deploy this folder to Netlify
├── data/
│   ├── 01_customer_tickets.csv           # Upload to tickets_inbox in Supabase
│   └── Knowledge Sources/               # Paste contents into knowledge_sources table
│       ├── 02_product_reference.txt
│       ├── 03a_rate_card_engraving.csv
│       ├── 03b_rate_card_servicing.csv
│       ├── 04_faq_document.txt
│       └── 05_SOP.txt
├── n8n/
│   ├── docker-compose.yml               # Local Docker alternative to n8n Cloud
│   ├── .env.example
│   └── workflows/
│       ├── WF1_Core_Intelligence_Loop.json
│       ├── WF2_Self_Improving_KB_Loop.json
│       └── WF3_Marketing_Insight_Generator.json
└── supabase/
    └── schema.sql                        # Run once in Supabase SQL Editor
```

---

## Built With

| Tool | Role |
|---|---|
| **n8n** | Workflow automation engine |
| **Qwen AI (qwen-plus)** | LLM for classification, KB search, reply drafting, insight generation |
| **Supabase** | PostgreSQL database + instant REST API |
| **Tavily** | Real-time web search for community sentiment |
| **Vanilla HTML/JS** | Dashboard — zero dependencies, opens in browser |

---

*Built by [Morning Wu](https://www.linkedin.com/in/morningwu/) for the e27 Echelon Singapore AI Workflow Competition 2026.*
*Founder of [AfterWork Startup](https://app.afterworkstartup.com) — helping 1M non-techies master practical AI skills.*
