# CLAUDE.md
 
## Project

"The Dashboard Nobody Trusts" — a CLV analytics platform built on AdventureWorks for Postgres.

Four stakeholders calculate Customer Lifetime Value four different ways. This project provides

the single source of truth, reconciliation engine, and exploratory tools via MCP.
 
## MCP Server

This project exposes an MCP server (`src/server.py`) with 5 tools for data access:
 
| Tool | Purpose |

|------|---------|

| `get_clv(method, customer_id?, limit?)` | Calculate CLV using revenue / gross_margin / net_margin / predictive |

| `reconcile(customer_id?, limit?)` | Compare all 4 methods side-by-side, explain divergence drivers |

| `query_data(sql)` | Run read-only SQL against AdventureWorks (SELECT only) |

| `list_customers(territory_id?, limit?)` | Browse customers with order counts and revenue |

| `what_if(freight_multiplier?, cost_multiplier?, limit?)` | Scenario modeling — adjust costs, see CLV impact |
 
### Running the MCP server

```bash

# Start database first

docker-compose up -d
 
# Install deps

pip install mcp psycopg2-binary
 
# The server runs over stdio — Claude Code connects to it directly

```
 
### Claude Code config (.mcp.json in project root)

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
 
## Key decisions (see docs/adr-001-clv-definition.md)

- "Canonical" CLV for exec reporting = gross margin (actual product economics, no shared-cost allocation)

- Freight/tax allocated per-order-header, not per-line-item (avoids double-counting)

- Customers with 1 order: lifespan floors to 1 year

- NULL StandardCost products: excluded from margin calcs, flagged in reconciliation
 
## Conventions

- Python 3.11+, psycopg2 for DB access

- All CLV functions return JSON with Decimal→float conversion

- SQL queries use parameterized inputs (no string formatting)

- Tests: `pytest tests/`
 
## The 4 CLV methods explained

1. **Revenue** (Sales VP): Sum of TotalDue — simple, inflated, ignores costs

2. **Gross margin** (Finance): LineTotal minus (StandardCost × Qty) — real product economics

3. **Net margin** (CFO): Gross margin minus freight and tax — most conservative

4. **Predictive** (Data Sci): (orders / lifespan) × AOV × lifespan — forward-looking, noisy for small samples
 
## What to ask me (example prompts)

- "Use the reconcile tool to find the top 10 customers with the biggest CLV disagreement"

- "Get the CLV for customer 29825 using all 4 methods"

- "What if we reduced freight costs by 30%? How does that change the top customer rankings?"

- "Query the data to find which territories have the most high-value customers"
 
Lests include in the Claude.md
 
That we would be using 3 agents dev, test data, and also that we would be leveraging existing reporting capabilities in the client 
 