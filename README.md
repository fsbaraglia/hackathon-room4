# The Dashboard Nobody Trusts
 
**One metric. Four numbers. Zero trust.**
 
Somewhere in this business there are 40 dashboards across three BI tools. Executives make decisions by gut because the numbers never match. Customer Lifetime Value — the metric everyone agrees matters most — is calculated four different ways depending on who you ask.
 
This project fixes that. We built a **semantic layer as an MCP server** that defines CLV once, exposes it as tools, and lets both humans and AI agents query, reconcile, and scenario-test the numbers through a single protocol. No more spreadsheet wars.
 
---
 
## The Problem
 
| Stakeholder | Their CLV formula | What it misses |
|---|---|---|
| **Sales VP** | Sum of `TotalDue` per customer | Ignores product costs entirely |
| **Finance** | `LineTotal - (StandardCost × Qty)` | Doesn't account for freight or tax |
| **CFO** | Gross margin minus freight and tax | Penalizes high-shipping territories unfairly |
| **Data Science** | Frequency × AOV × predicted lifespan | Extrapolates wildly from small samples |
 
The same customer — C-29825 — is worth **$45,200** or **$12,100** depending on which dashboard you open. Both numbers are defensible. Neither is complete.
 
The real problem isn't that people are wrong. It's that **the metric lives in BI SQL views that nobody owns, nobody tests, and nobody can explain.** When the VP asks "why did CLV drop on Tuesday?" the answer is a 45-minute email thread, not a query.
 
---
 
## The Solution
 
We moved the metric definition out of BI tools and into **code** — testable, versioned, explainable Python functions — and exposed them through the **Model Context Protocol (MCP)**. This means:
 
- **Claude Code can query CLV directly** by calling MCP tools, no copy-paste from dashboards
- **The reconciliation is a tool call**, not a manual spreadsheet exercise
- **What-if scenarios** (lower freight, higher costs) run in seconds, not days
- **Every calculation is traceable** back to a documented definition with edge cases handled
 
### Why MCP?
 
MCP (Model Context Protocol) solves the fundamental integration problem that created the 40-dashboard mess in the first place. Here's what it addresses:
 
**Before MCP — the old world:**
- Metric definitions are buried in SQL views inside BI tools (Tableau, Power BI, Looker)
- Each tool has its own connection to the database, its own caching, its own version of the query
- When someone asks a question, they either open the "right" dashboard (which one?) or ask an analyst to write a new query
- AI assistants can't access the data at all — they can only summarize what humans paste into chat
- No audit trail of who calculated what, when, or with which assumptions
 
**After MCP — what we built:**
- One semantic layer defines CLV four ways, documents every assumption, and handles every edge case
- Claude Code connects to it natively via stdio — zero HTTP glue, zero REST wrappers
- Any MCP client (Claude Code, a dashboard, a CI pipeline) calls the same tools with the same definitions
- `query_data()` provides a read-only SQL escape hatch for ad-hoc questions, with write protection built in
- Every tool call is structured, parameterized, and loggable — the VP can see exactly which method was used
 
**The specific problems MCP solves for us:**
 
| Problem | How MCP addresses it |
|---|---|
| 4 conflicting CLV formulas | All 4 live in `get_clv(method=...)` — same code, same data, explicit choice |
| "Which dashboard do I open?" | One entry point: call `reconcile()` and see all 4 side-by-side |
| No one can explain the divergence | `reconcile()` returns `divergence_drivers` explaining *why* numbers differ |
| Analysts are the bottleneck | Claude Code calls tools directly — natural language in, structured data out |
| What-if analysis takes days | `what_if(freight_multiplier=0.8)` runs in seconds |
| No tests, no versioning | Python functions with pytest, in a git repo, with CI |
| Dashboard sprawl | Dashboards become thin views over MCP tools, not independent data silos |
 
---
 
## Architecture
 
```
┌─────────────────────────────────────────────────────────────┐
│                        MCP Clients                          │
│                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│   │ Claude Code   │    │ Streamlit    │    │ CI / pytest  │  │
│   │ (NL queries)  │    │ (Dashboard)  │    │ (Validation) │  │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
└──────────┼───────────────────┼───────────────────┼──────────┘
           │                   │                   │
           │         MCP Protocol (stdio)          │
           │                   │                   │
┌──────────┴───────────────────┴───────────────────┴──────────┐
│                    MCP Server (Python)                       │
│                                                             │
│   ┌────────────┐ ┌────────────┐ ┌────────────┐             │
│   │ get_clv()  │ │ reconcile()│ │query_data()│             │
│   │ 4 methods  │ │ compare +  │ │ read-only  │             │
│   │            │ │ explain    │ │ SQL access  │             │
│   └────────────┘ └────────────┘ └────────────┘             │
│   ┌─────────────────┐ ┌─────────────────┐                  │
│   │list_customers() │ │   what_if()     │                  │
│   │ filter/browse   │ │ scenario model  │                  │
│   └─────────────────┘ └─────────────────┘                  │
│                                                             │
│   Semantic Layer: documented assumptions, edge case         │
│   handling, Decimal precision, parameterized queries        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                    psycopg2 (TCP)
                          │
┌─────────────────────────┴───────────────────────────────────┐
│              PostgreSQL — AdventureWorks                     │
│   68 tables · 5 schemas · 31K orders · 20K customers        │
│   sales · production · purchasing · person · humanresources │
└─────────────────────────────────────────────────────────────┘
```
 
### MCP Tools Reference
 
| Tool | Parameters | Returns |
|---|---|---|
| `get_clv` | `method` (revenue/gross_margin/net_margin/predictive), `customer_id?`, `limit?` | CLV per customer for selected method |
| `reconcile` | `customer_id?`, `limit?` | All 4 CLV values + spread + divergence drivers |
| `query_data` | `sql` (SELECT only) | Raw query results (max 100 rows) |
| `list_customers` | `territory_id?`, `limit?` | Customers with order counts and revenue |
| `what_if` | `freight_multiplier?`, `cost_multiplier?`, `limit?` | Adjusted net margin CLV under scenario |
 
---
 
## Getting Started
 
### Prerequisites
- Docker and Docker Compose
- Python 3.11+
- Claude Code (for MCP integration)
 
### 1. Start the database
```bash
git clone https://github.com/lorint/AdventureWorks-for-Postgres.git
cd AdventureWorks-for-Postgres
docker-compose up -d
```
 
### 2. Install dependencies
```bash
pip install -r requirements.txt
```
 
### 3. Run the MCP server
```bash
python src/server.py
```
 
### 4. Connect Claude Code
The `.mcp.json` in the project root auto-configures the connection:
```json
{
  "mcpServers": {
    "clv-analytics": {
      "command": "python",
      "args": ["src/server.py"],
      "env": {
        "DB_NAME": "Adventureworks",
        "DB_USER": "postgres",
        "DB_PASS": "postgres",
        "DB_HOST": "localhost",
        "DB_PORT": "5432"
      }
    }
  }
}
```
 
### 5. Try it
In Claude Code:
```
> Use the reconcile tool to find the top 10 customers with the biggest CLV disagreement
> Get the CLV for customer 29825 using all 4 methods
> What if we reduced freight costs by 30%? How does that change the top customer rankings?
```
 
### 6. Launch the dashboard
```bash
cd dashboard
streamlit run app.py
```
 
---
 
## Key Decisions
 
See [docs/adr-001-clv-definition.md](docs/adr-001-clv-definition.md) for full rationale.
 
**Canonical CLV = Gross Margin.** Reflects real product economics without over-allocating shared costs. Revenue overstates value; net margin penalizes geography; predictive extrapolates noise.
 
**Freight/tax allocated per order header, not per line item.** The AdventureWorks schema stores freight and tax on `salesorderheader`, not `salesorderdetail`. Joining without deduplication double-counts — we use a CTE to aggregate line-level margins first, then subtract per-order overhead.
 
**Lifespan floors to 1 year.** A customer with one order has a lifespan of 0 in raw data. Predictive CLV divides by lifespan — flooring to 1 prevents division by zero and treats one-time buyers as having at least a one-year relationship.
 
**NULL StandardCost products are flagged, not dropped.** 12 products in AdventureWorks lack cost data. Silently excluding them understates margins. Our reconciliation tool flags these in the divergence drivers so stakeholders know the gap exists.
 
---
 
## Road to A2A: What's Next
 
Our MCP server is deliberately designed as a stepping stone toward the **Agent2Agent (A2A) Protocol** — Google's open standard for inter-agent collaboration, now at v0.3 under the Linux Foundation with 150+ partner organizations.
 
### Why we're planning for A2A
 
MCP solves the **tool access** problem: AI connects to data through structured tools. But as analytics teams scale, the problem shifts from "how does the AI access data?" to **"how do specialized agents collaborate on complex analytical tasks?"**
 
Consider this scenario: a VP asks "Why did CLV drop in the Northwest territory last quarter, and what should we do about it?" Answering that well requires:
 
1. A **data agent** that owns the database connection, schema knowledge, and query optimization
2. An **analytics agent** that owns the CLV definitions, reconciliation logic, and statistical models
3. A **recommendation agent** that interprets the analysis and suggests business actions
4. A **presentation agent** that formats the output for the audience (exec summary vs. analyst deep-dive)
 
Today, our MCP server bundles all of this into one process. That works at hackathon scale. It doesn't work when:
- Different teams own different agents and need to evolve them independently
- Agents need to discover each other dynamically (who can calculate CLV? who knows about territory performance?)
- The conversation between agents requires back-and-forth negotiation, not just tool calls
- Security boundaries require agents to collaborate without sharing internal state
 
### How A2A addresses this
 
| MCP (what we built today) | A2A (where we're headed) |
|---|---|
| Tools exposed via stdio | Agents communicate via JSON-RPC 2.0 over HTTPS |
| Single server, single process | Independent agents discover each other via Agent Cards |
| Client calls tools directly | Agents delegate tasks and negotiate capabilities |
| Synchronous request/response | Supports async, streaming (SSE), and push notifications |
| All logic co-located | Agents preserve opacity — no shared memory or internal state |
 
### Our A2A migration path
 
The MCP server we built today maps cleanly to A2A agent boundaries:
 
```
Today (MCP — single server):
┌──────────────────────────────────────────┐
│            MCP Server                    │
│  get_clv · reconcile · query_data        │
│  list_customers · what_if                │
└──────────────────────────────────────────┘
 
Tomorrow (A2A — agent ecosystem):
┌─────────────────┐     ┌─────────────────┐
│  Data Agent      │────▶│ Analytics Agent │
│  Agent Card:     │ A2A │ Agent Card:     │
│  - query_data    │◀────│ - get_clv       │
│  - list_customers│     │ - reconcile     │
│  - schema info   │     │ - what_if       │
└─────────────────┘     └────────┬────────┘
                                 │ A2A
                        ┌────────▼────────┐
                        │ Insights Agent  │
                        │ Agent Card:     │
                        │ - explain_trend │
                        │ - recommend     │
                        │ - format_report │
                        └─────────────────┘
```
 
Each agent publishes an **Agent Card** (a JSON file at `/.well-known/agent.json`) describing its capabilities, supported authentication, and endpoint. The orchestrating agent discovers available agents, delegates tasks, and assembles the final response — all without any agent needing access to another's internal logic.
 
### What we'd build first
 
1. **Split the MCP server into two A2A agents** — a Data Agent (owns DB access) and an Analytics Agent (owns CLV logic). This tests the protocol boundary without changing the math.
2. **Add Agent Cards** with capability descriptions so agents can be discovered dynamically.
3. **Implement task lifecycle** — the analytics agent creates a task when CLV is requested, streams progress for long-running reconciliations, and returns artifacts on completion.
4. **Add an Insights Agent** that takes reconciliation output and generates natural-language explanations of trends and recommendations.
 
### MCP + A2A complement each other
 
This isn't MCP *or* A2A — it's both. As the official A2A documentation recommends: **use MCP for tools, use A2A for agents.** Our Data Agent would still use MCP internally to connect to Postgres. The A2A layer sits on top, enabling agent-to-agent collaboration that MCP wasn't designed for.
 
---
 
## Testing
 
Seven automated tests covering real edge cases found in AdventureWorks data:
 
| Test | Category | What it verifies |
|---|---|---|
| `test_all_methods_return_data` | Evaluation | Every CLV method returns 100+ customers |
| `test_revenue_gte_gross_margin` | Evaluation | Revenue CLV >= Gross Margin CLV always (costs can't be negative) |
| `test_single_order_no_division_by_zero` | Edge thinking | Predictive CLV handles lifespan=0 by flooring to 1 |
| `test_null_standard_cost_excluded` | Edge thinking | Products without cost data are flagged, not silently dropped |
| `test_reconcile_flags_divergence` | Evaluation | `reconcile()` returns drivers for high-spread customers |
| `test_query_data_blocks_writes` | Edge thinking | `query_data()` rejects INSERT/UPDATE/DELETE/DROP |
| `test_what_if_lower_freight` | Reversal thinking | Reducing freight always increases net margin CLV |
 
Run with:
```bash
pytest tests/ -v
```
 
---
 
## Project Structure
 
```
hackathon-room4/
├── CLAUDE.md                      # How Claude Code works with this project
├── README.md                      # This file
├── presentation.html              # 5-minute pitch deck
├── .mcp.json                      # Claude Code MCP server configuration
├── docker-compose.yml             # Postgres + AdventureWorks
├── requirements.txt               # Python dependencies
├── docs/
│   └── adr-001-clv-definition.md  # Architecture Decision Record
├── src/
│   └── server.py                  # MCP server — the semantic layer
├── tests/
│   └── test_clv.py                # Edge case and evaluation tests
└── dashboard/
    └── app.py                     # Streamlit dashboard
```
 
---
 
## Team
 
**Room 4** — Built with Claude Code at the APAC CC Workshop Hackathon.
 
The entire MCP server, dashboard, presentation, and test suite were developed in a 2-hour sprint using Claude Code as a pair programming partner. The `CLAUDE.md` file documents how we taught the tool to understand our domain, our data, and our conventions.
