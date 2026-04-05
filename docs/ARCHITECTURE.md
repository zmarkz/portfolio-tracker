# Portfolio Intelligence Platform — Architecture

> **Last updated**: 2026-04-06 | **Version**: 4.0 | **Author**: Claude Code + Markandey Singh

---

## 1. System Overview

An AI-first personal finance advisor for Indian retail investors. Multi-broker, multi-application platform with portfolio management across 7 Indian brokers, XIRR performance analytics, goal-based planning, tax optimization (FIFO + harvesting), net worth tracking, CDSL/NSDL CAS parsing, daily AI briefings, and document upload (Form 16, 26AS, AIS) — all orchestrated through Agent Farm.

| Metric | Value |
|--------|-------|
| Applications | 5 (2 SPAs, 1 API, 1 Gateway, 1 Orchestrator) |
| MySQL Tables | 25+ (business data) |
| PostgreSQL Tables | 7 (MCP infrastructure) |
| MCP Servers | 4 (portfolio, web-search, aws, atlassian) |
| MCP Tools | 17 registered |
| Agent Templates | 6 |
| Supported Brokers | 7 (Zerodha, Upstox, Angel One, 5paisa, ICICI Direct, Groww, Paytm Money) |
| Data Import | CDSL CAS, NSDL CAS, MFCentral, Broker CSV, Form 16, 26AS, AIS |
| Languages | Java 23, TypeScript 5.9 |
| Databases | MySQL 8 (business), PostgreSQL 15 (infra), Redis 7 (queue) |
| API Endpoints | 75+ REST + 1 WebSocket + 2 MCP protocol |
| Frontend Pages | 13 (+ login) |
| Scheduled Jobs | 3 (performance 4:30PM, tax harvest Sat 9AM, briefing 6AM) |

### Test User

| Field | Value |
|-------|-------|
| Name | Rahul Sharma |
| Email | `testuser@portfolio.ai` |
| Password | `Test@123` |
| Portfolios | 2 (Long Term Wealth: 10 stocks, Smallcap Bets: 3 stocks) |
| Net Worth | ~₹30.4L (equity + MF + FD + PPF + Gold + Savings) |
| Risk Profile | MODERATE (7/10) |
| Test Guide | See `docs/TEST_CASES.md` |

---

## 2. High-Level Architecture

```
                                ┌──────────────┐
                                │   BROWSER    │
                                └──────┬───────┘
                                       │
                                ┌──────▼───────┐
                                │    NGINX     │ :3000
                                │  /        → Frontend (:5173)
                                │  /api/*   → Backend  (:8080)
                                │  /ws      → WebSocket(:8080)
                                │  /admin/* → Admin SPA(:5174)
                                │  /gateway → MCP GW   (:9080)
                                │  /agent   → Agent Farm(:8082)
                                └──┬──┬──┬──┬──┘
               ┌───────────────────┘  │  │  └──────────────────┐
               │                      │  │                     │
      ┌────────▼────────┐    ┌───────▼──▼───────┐    ┌───────▼─────────┐
      │  Portfolio       │    │  Portfolio       │    │  Admin Nexus    │
      │  Frontend        │    │  Tracker API     │    │  (React SPA)   │
      │  React (:5173)   │    │  Spring Boot     │    │  :5174         │
      │                  │    │  (:8080)         │    └───────┬─────────┘
      │ - Dashboard      │    │                  │            │
      │ - Holdings       │    │ - Auth (JWT)     │            │
      │ - AI Assessment  │    │ - Portfolios     │            │
      │ - Tax Dashboard  │    │ - AI Analysis    │    ┌───────▼─────────┐
      │ - Income/Trades  │    │ - Tax Engine     │    │  MCP Gateway    │
      │ - Transactions   │    │ - Income CRUD    │    │  Fastify        │
      └─────────┬────────┘    │ - Approvals      │    │  :9080/:8081    │
                │             │ - Kite Sync      │    │                 │
                │             └────┬──────┬──────┘    │ - Tool Registry │
                │                  │      │           │ - Consumer Auth │
                │                  │      │           │ - Circuit Breaker│
                ▼                  │      │           └────────┬────────┘
           ┌─────────┐            │      │                    │
           │ MySQL 8 │            │  ┌───▼──────────┐  ┌─────┼──────────────┐
           │ :3306   │            │  │ Agent Farm   │  │     │              │
           └─────────┘            │  │ Fastify      │  │  ┌──▼──┐  ┌──────▼───┐
                                  │  │ :8082        │  │  │PT   │  │Web Search│
                                  │  │              │  │  │MCP  │  │MCP       │
                                  │  │ - LLM Calls  │  │  │:3004│  │:3001     │
                                  │  │ - BullMQ     │  │  └─────┘  └──────────┘
                                  │  │ - Streaming  │  │
                                  │  └──────┬───────┘  │
                                  │         │          │
                             ┌────▼─────┐ ┌─▼──────┐ ┌─▼──────────┐
                             │Anthropic │ │ Redis  │ │ PostgreSQL │
                             │Claude API│ │ :6379  │ │ :5432      │
                             │(External)│ └────────┘ └────────────┘
                             └──────────┘
```

---

## 3. Service Catalog

| # | Service | Type | Stack | Port | Container | Purpose |
|---|---------|------|-------|------|-----------|---------|
| 1 | Portfolio Frontend | React SPA | React 19, Vite, TailwindCSS, Zustand | 5173 | `portfolio_tracker_frontend` | User dashboard |
| 2 | Portfolio Tracker API | REST API | Java 23, Spring Boot 3.2, JPA, Maven | 8080 | `portfolio_tracker_api` | Business logic |
| 3 | Admin Nexus | React SPA | React 19, Vite, TailwindCSS, Recharts | 5174 | `admin_nexus` | MCP Farm admin |
| 4 | MCP Gateway | API + MCP | Node.js, Fastify, Drizzle, MCP SDK | 9080/8081 | `mcp_gateway` | Tool routing |
| 5 | Agent Farm | Orchestrator | Node.js, Fastify, Vercel AI SDK, BullMQ | 8082 | `agent_farm` | AI execution |
| 6 | Portfolio MCP | MCP Server | Node.js, TypeScript, MCP SDK | 3004 | `mcp_portfolio_tracker` | Portfolio tools |
| 7 | Web Search MCP | MCP Server | Node.js, TypeScript, MCP SDK | 3001 | `mcp_web_search` | Search tools |
| 8 | Nginx | Reverse Proxy | Nginx | 3000 | `platform_nginx` | Routing |
| 9 | Redis | Queue/Cache | Redis 7 Alpine | 6379 | `platform_redis` | BullMQ |
| 10 | MySQL | Database | MySQL 8.0 | 3306 | Host | Business data |
| 11 | PostgreSQL | Database | PostgreSQL 15 | 5432 | Host | MCP metadata |

---

## 4. Authentication Architecture

### 7 Layers of Authentication

```
Layer 1: User Auth (JWT)
  Who:   End users (portfolio owners)
  How:   POST /api/auth/login → JWT (HS256, 24hr expiry)
  Where: Authorization: Bearer <jwt>
  Check: JwtTokenFilter on every request

Layer 2: Service-to-Service
  Who:   Portfolio API → Agent Farm
  How:   AGENT_FARM_API_KEY (env var)
  Where: Authorization: Bearer <api-key>

Layer 3: MCP Consumer
  Who:   Agent Farm → MCP Gateway
  How:   Consumer API key from consumers table
  Where: Authorization: Bearer <consumer-key>
  Check: Subscription ACL (canRead + canExecute)

Layer 4: Admin
  Who:   Admin Nexus → Gateway + Agent Farm
  How:   ADMIN_API_KEY (default: "admin-secret")
  Where: Authorization: Bearer <admin-key>

Layer 5: MCP Server Auth
  Who:   Portfolio MCP → Portfolio API
  How:   Service account JWT (login on startup)

Layer 6: Ownership Validation
  Who:   Authenticated user → only their data
  How:   verifyPortfolioOwnership() in AIController
  Check: SecurityContextHolder → userId == portfolio.userId

Layer 7: External APIs
  Anthropic: x-api-key header
  Zerodha:   OAuth 2.0 flow
```

---

## 5. Data Flow: AI Portfolio Analysis

```
Browser                 Portfolio API          Agent Farm         MCP Gateway        Claude API
  │                          │                     │                  │                  │
  │ POST /api/ai/            │                     │                  │                  │
  │ portfolio/1/analyze      │                     │                  │                  │
  │─────────────────────────>│                     │                  │                  │
  │                          │                     │                  │                  │
  │                 1. Verify Ownership             │                  │                  │
  │                    (JWT → userId check)         │                  │                  │
  │                          │                     │                  │                  │
  │                 2. Build Portfolio Context      │                  │                  │
  │                    (holdings, P&L, sectors)     │                  │                  │
  │                          │                     │                  │                  │
  │                 3. Check Agent Farm             │                  │                  │
  │                    GET /health (3s timeout)     │                  │                  │
  │                          │────────────────────>│                  │                  │
  │                          │                     │                  │                  │
  │                    ┌─────┴── Available? ───┐   │                  │                  │
  │                    │ YES                   │NO │                  │                  │
  │                    │                       │   │                  │                  │
  │                    │ POST /api/tasks       │   │                  │                  │
  │                    │ {templateId:4,        │   │                  │                  │
  │                    │  mode:"sync"}         │   │                  │                  │
  │                    │──────────────────────>│   │                  │                  │
  │                    │                       │   │                  │                  │
  │                    │            LLM + tools│   │  Direct Claude   │                  │
  │                    │                       │   │  API call with   │                  │
  │                    │  Calls MCP tools:     │   │  inline JSON     │                  │
  │                    │  portfolio-tracker__*  │   │  schema          │                  │
  │                    │  web-search__*         │   │                  │                  │
  │                    │                       │   │──────────────────────────────────>│
  │                    │  Structured JSON       │   │                  │                  │
  │                    │<─────────────────────│   │  Structured JSON │                  │
  │                    │                       │   │<──────────────────────────────────│
  │                    └───────────────────────┘   │                  │                  │
  │                          │                     │                  │                  │
  │                 4. Save to analysis_history     │                  │                  │
  │                    (model, prompt, response)    │                  │                  │
  │                          │                     │                  │                  │
  │  {healthScore: 82,       │                     │                  │                  │
  │   riskProfile: "Moderate",                     │                  │                  │
  │   risks: [...],          │                     │                  │                  │
  │   recommendations: [...]}│                     │                  │                  │
  │<─────────────────────────│                     │                  │                  │
```

---

## 6. Data Flow: Tax Calculation (FIFO Lot Matching)

```
Input: Financial Year "2025-26" (April 2025 → March 2026)

Step 1: Fetch SELL trades in FY
  ┌──────────────────────────────────────────────────┐
  │ SELL  RELIANCE  2025-06-10  10 shares @ ₹2,900  │
  │ SELL  TCS       2025-11-20   5 shares @ ₹3,800  │
  └──────────────────────────────────────────────────┘

Step 2: Fetch ALL BUY trades for sold symbols (FIFO queue)
  ┌──────────────────────────────────────────────────┐
  │ BUY   RELIANCE  2025-01-15  10 shares @ ₹2,650  │
  │ BUY   TCS       2025-03-20   5 shares @ ₹3,450  │
  └──────────────────────────────────────────────────┘

Step 3: FIFO Match (earliest buy consumed first)
  ┌──────────────────────────────────────────────────┐
  │ RELIANCE: BUY 2025-01-15 → SELL 2025-06-10      │
  │   Holding: 146 days (< 365) → STCG              │
  │   Gain: (₹2,900 - ₹2,650.50) × 10 = ₹2,495    │
  │                                                   │
  │ TCS: BUY 2025-03-20 → SELL 2025-11-20           │
  │   Holding: 245 days (< 365) → STCG              │
  │   Gain: (₹3,800 - ₹3,450) × 5 = ₹1,750         │
  └──────────────────────────────────────────────────┘

Step 4: Apply Indian Tax Rules
  ┌──────────────────────────────────────────────────┐
  │ LTCG (> 365 days):  12.5% above ₹1.25L exempt   │
  │   Total: ₹0 → Tax: ₹0                           │
  │                                                   │
  │ STCG (≤ 365 days): 20% flat                     │
  │   Total: ₹4,245 → Tax: ₹849                     │
  └──────────────────────────────────────────────────┘

Step 5: Aggregate Income
  ┌──────────────────────────────────────────────────┐
  │ Dividends:    ₹5,000   (TDS: ₹500)              │
  │ FD Interest:  ₹12,000  (TDS: ₹1,200)            │
  │ Total Income: ₹21,245  Total TDS: ₹1,700        │
  └──────────────────────────────────────────────────┘
```

### Indian Tax Rules (Post Budget 2024)

| Asset Class | Holding Period | Tax Rate | Exemption |
|-------------|---------------|----------|-----------|
| Listed Equity (STT paid) | > 12 months | 12.5% LTCG | ₹1.25 Lakh/FY |
| Listed Equity (STT paid) | ≤ 12 months | 20% STCG | None |
| Debt Mutual Fund | Any | Slab rate | No indexation |
| SGB (maturity) | At maturity | 0% | Tax-free |
| Dividends | N/A | Slab rate | TDS u/s 194 |
| FD Interest | N/A | Slab rate | TDS u/s 194A |

---

## 7. Data Flow: MCP Tool Execution

```
Agent Farm          MCP Gateway           Client Pool          MCP Server         Portfolio API
  │                     │                     │                    │                   │
  │ POST /api/tools/    │                     │                    │                   │
  │ call {tool:         │                     │                    │                   │
  │  "portfolio-tracker │                     │                    │                   │
  │   __get_holdings",  │                     │                    │                   │
  │  args:{userId:1}}   │                     │                    │                   │
  │────────────────────>│                     │                    │                   │
  │                     │ 1. Parse tool path  │                    │                   │
  │                     │    mcp: "portfolio- │                    │                   │
  │                     │     tracker"        │                    │                   │
  │                     │    tool: "get_      │                    │                   │
  │                     │     holdings"       │                    │                   │
  │                     │                     │                    │                   │
  │                     │ 2. Check consumer   │                    │                   │
  │                     │    subscription     │                    │                   │
  │                     │    (canExecute)     │                    │                   │
  │                     │                     │                    │                   │
  │                     │ 3. Route to pool    │                    │                   │
  │                     │────────────────────>│ Lazy init client  │                   │
  │                     │                     │───────────────────>│                   │
  │                     │                     │   JSON-RPC call    │                   │
  │                     │                     │                    │ JWT auth + GET    │
  │                     │                     │                    │ /api/portfolios/  │
  │                     │                     │                    │ 1/holdings        │
  │                     │                     │                    │──────────────────>│
  │                     │                     │                    │  Holdings JSON    │
  │                     │                     │                    │<──────────────────│
  │                     │                     │  Tool result       │                   │
  │                     │                     │<───────────────────│                   │
  │                     │<────────────────────│                    │                   │
  │                     │                     │                    │                   │
  │                     │ 4. Audit log        │                    │                   │
  │                     │    (tool_calls tbl) │                    │                   │
  │  Tool result JSON   │                     │                    │                   │
  │<────────────────────│                     │                    │                   │
```

---

## 8. Database Schema

### MySQL — `portfolio` (23 tables)

```
users ─────────┬─── portfolios ──────┬─── portfolio_holdings ──── stocks
  │             │                     │
  │             │                     └─── transactions
  │             │
  ├─── user_preferences (1:1)        ┌─── stock_price_history
  ├─── income_entries                 │
  ├─── tradebook_entries              stocks ────┘
  ├─── analysis_history
  ├─── chat_history                   financial_goals ──── goal_allocations
  ├─── approval_requests
  ├─── financial_goals                net_worth_assets
  ├─── net_worth_assets               mutual_fund_holdings
  ├─── mutual_fund_holdings           performance_snapshots
  ├─── notifications                  tax_harvest_opportunities
  ├─── daily_briefings                daily_briefings
  ├─── watchlists ──── watchlist_items
  └─── alerts
```

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| `users` | id, email, password, role (USER/ADMIN), kite_token | Authentication |
| `portfolios` | id, user_id, name, currency | Portfolio containers |
| `portfolio_holdings` | portfolio_id, stock_id, quantity, avg_buy_price | Current positions |
| `stocks` | symbol, company_name, sector, current_price | Stock master |
| `transactions` | portfolio_id, stock_id, type (BUY/SELL), quantity, price | Trade log |
| `stock_price_history` | stock_id, date, open, high, low, close | Historical prices |
| `alerts` | user_id, stock_id, condition, threshold | Price alerts |
| `income_entries` | user_id, type, amount, financial_year, tax_deducted | Investment income |
| `tradebook_entries` | user_id, symbol, trade_date, trade_type, price, order_id | Kite trade history |
| `analysis_history` | user_id, portfolio_id, type, ai_model, raw_response, health_score | AI analysis audit |
| `chat_history` | user_id, portfolio_id, role, content | Chat persistence |
| `approval_requests` | user_id, portfolio_id, status, payload, review_notes | Approval workflow |

### PostgreSQL — `mcp_farm` (7 tables)

| Table | Purpose |
|-------|---------|
| `mcp_servers` | Registered MCP server endpoints and health |
| `tool_definitions` | Cached tool schemas (name, description, inputSchema) |
| `consumers` | API consumers with keys and rate limits |
| `subscriptions` | Consumer → MCP server access (canRead/canExecute) |
| `tool_calls` | Audit log of every tool execution |
| `agent_templates` | LLM configs (model, system prompt, MCP associations) |
| `agent_tasks` | Task execution records (input, output, status, time) |

---

## 9. MCP Tool Registry

| MCP Server | Tool | User-Scoped | Description |
|------------|------|-------------|-------------|
| portfolio-tracker (:3004) | `list_portfolios` | Yes | List user portfolios |
| | `get_portfolio_holdings` | Yes | Holdings with gain/loss |
| | `get_portfolio_metrics` | Yes | Sector allocation, top holdings |
| | `get_portfolio_summary` | Yes | Full portfolio view |
| | `get_portfolio_transactions` | Yes | Transaction history |
| | `get_recent_transactions` | Yes | Recent trades |
| | `search_stocks` | No | Symbol/name search |
| | `get_stock_price` | No | Current NSE price |
| | `get_stock_analysis` | No | Technical indicators |
| | `get_tax_summary` | Yes | LTCG/STCG breakdown |
| | `get_income_entries` | Yes | Recorded income |
| web-search (:3001) | `web_search` | No | Brave/DuckDuckGo search |
| | `get_page_content` | No | Extract text from URL |

---

## 10. Agent Templates

| ID | Name | Model | MCP Servers | Output |
|----|------|-------|-------------|--------|
| 1 | Portfolio Analyst | claude-sonnet-4-6 | portfolio-tracker, web-search | Free-form text |
| 2 | Research Agent | claude-3.5-sonnet | web-search | Research synthesis |
| 3 | DevOps Agent | claude-3.5-sonnet | aws, atlassian | Infrastructure |
| 4 | Portfolio Analysis | claude-sonnet-4-6 | portfolio-tracker, web-search | Structured JSON (healthScore, risks, recommendations) |
| 5 | Rebalancing Agent | claude-sonnet-4-6 | portfolio-tracker, web-search | JSON array (current/target weights) |
| 6 | Tax Advisor | claude-sonnet-4-6 | portfolio-tracker | Indian tax optimization |

---

## 11. API Endpoint Map (30+ endpoints)

### Auth (Public)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/register` | Register new user |
| POST | `/api/auth/login` | Login → JWT token |

### Portfolio Management
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolios` | List user's portfolios |
| POST | `/api/portfolios` | Create portfolio |
| GET | `/api/portfolios/{id}` | Get portfolio by ID |
| GET | `/api/portfolios/{id}/holdings` | Get holdings |
| POST | `/api/portfolios/{id}/holdings` | Add holding |
| DELETE | `/api/portfolios/{pid}/holdings/{hid}` | Remove holding |
| GET | `/api/portfolios/{id}/analysis` | Portfolio analysis view |
| POST | `/api/portfolios/{id}/holdings/upload` | Upload CSV/XLSX |

### Stocks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stocks/search` | Search stocks |
| GET | `/api/stocks/{symbol}/price` | Current price |
| GET | `/api/stocks/{symbol}/analysis` | Technical indicators |

### Transactions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolios/{id}/transactions` | Transaction history |
| POST | `/api/transactions` | Execute buy/sell |

### AI Analysis (Ownership Protected)
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/ai/portfolio/{id}/analyze` | Full AI analysis |
| POST | `/api/ai/portfolio/{id}/chat` | Chat (SSE streaming) |
| GET | `/api/ai/portfolio/{id}/rebalance` | Rebalancing suggestions |
| GET | `/api/ai/portfolio/{id}/chat/history` | Chat history |
| POST | `/api/ai/portfolio/{id}/chat/history` | Save chat |
| DELETE | `/api/ai/portfolio/{id}/chat/history` | Clear chat |
| GET | `/api/ai/portfolio/{id}/analysis/history` | Analysis history |
| GET | `/api/ai/portfolio/{id}/analysis/latest` | Latest analysis |
| GET | `/api/ai/portfolio/{id}/rebalancing/history` | Rebalancing history |
| GET | `/api/ai/portfolio/{id}/rebalancing/latest` | Latest rebalancing |
| GET | `/api/ai/analysis/{analysisId}` | Full transparency detail |
| GET | `/api/ai/portfolio/{pid}/stock/{symbol}` | Stock recommendation |
| GET | `/api/ai/status` | AI config status |

### Tax & Tradebook (NEW)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tax/{fy}/summary` | Capital gains tax summary |
| POST | `/api/tax/tradebook/upload` | Upload Kite tradebook CSV |
| GET | `/api/tax/tradebook` | List tradebook entries |

### Income Tracking (NEW)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/income` | List income entries |
| POST | `/api/income` | Add income entry |
| DELETE | `/api/income/{id}` | Delete income entry |

### Approval Workflow (NEW)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/approvals` | List approval requests |
| GET | `/api/approvals/{id}` | Get approval detail |
| POST | `/api/approvals/{id}/approve` | Approve request |
| POST | `/api/approvals/{id}/reject` | Reject request |

### Zerodha Kite
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/kite/login` | Redirect to Zerodha OAuth |
| GET | `/api/kite/callback` | OAuth callback |
| POST | `/api/kite/connect` | Store access token |
| GET | `/api/kite/status` | Connection status |
| POST | `/api/kite/sync` | Sync holdings |

### WebSocket & Docs
| Path | Description |
|------|-------------|
| `/ws` | STOMP over SockJS (subscribe: `/topic/prices`) |
| `/swagger-ui.html` | Interactive API docs |
| `/v3/api-docs` | OpenAPI 3.0 spec |

---

## 12. Frontend Pages

| Route | Page | Features |
|-------|------|----------|
| `/login` | Login/Register | Email + password auth |
| `/dashboard` | Dashboard | Holdings table, metrics bar, sector chart, CSV import, Kite sync |
| `/holdings` | Holdings | Stock list, click through to analysis |
| `/performance` | **Performance** | XIRR cards, portfolio vs benchmark chart, period selector |
| `/stocks/:symbol` | Stock Analysis | Price, SMA50/200, RSI, MACD, AI insights |
| `/transactions` | Transactions | Buy/sell execution, history |
| `/ai` | AI Assessment | Chat (SSE), Analysis (JSON), Rebalance, History, Transparency |
| `/goals` | **Goals** | Goal cards, progress bars, SIP calculator, 3-scenario projections |
| `/networth` | **Net Worth** | Multi-asset total, donut chart, asset CRUD, liquidity breakdown |
| `/watchlist` | **Watchlist** | Stocks with target prices, alerts, live price comparison |
| `/tax` | Tax Dashboard | FY selector, LTCG/STCG cards, capital gains table, income breakdown |
| `/income` | Income & Trades | Income CRUD (7 types), Tradebook CSV upload, tabbed UI |
| `/settings` | **Settings** | Risk questionnaire (10-step), preferences, notification toggles |

---

## 13. Resilience Patterns

| Pattern | Implementation |
|---------|---------------|
| AI Fallback | Agent Farm first → Direct Claude API if unavailable |
| Circuit Breaker | MCP Gateway: 5 failures → 60s open → half-open retry |
| Connection Pool | Lazy init per MCP server, stale session eviction |
| Task Queue | BullMQ: 5 workers, 3 retries, exponential backoff |
| Agent Limits | 120s timeout, max 10 tool-use steps |
| Data Import Fallback | Kite API → CSV upload → Paste CSV text |
| Health Monitoring | 30s health check loop for all MCP servers |

---

## 14. Approval Workflow State Machine

```
              ┌─────────┐
 AI generates │ PENDING │
 suggestion   └────┬────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
   ┌────▼────┐     │    ┌────▼─────┐
   │APPROVED │     │    │REJECTED  │
   └────┬────┘     │    └──────────┘
        │          │
   ┌────▼────┐  ┌──▼──────┐
   │EXECUTED │  │ EXPIRED │
   │(Future) │  │ (TTL)   │
   └─────────┘  └─────────┘
```

---

## 15. Deployment

### Local Development (Docker Compose)
```
Host Machine (macOS)
├── MySQL 8 (:3306)           ← native
├── PostgreSQL 15 (:5432)     ← native
└── Docker Desktop
    └── platform_network (bridge)
        ├── redis (:6379)
        ├── nginx (:3000)
        ├── portfolio-tracker (:8080)
        ├── portfolio-tracker-frontend (:5173)
        ├── mcp-gateway (:9080/:8081)
        ├── agent-farm (:8082)
        ├── mcp-portfolio-tracker (:3004)
        ├── mcp-web-search (:3001)
        └── admin-nexus (:5174)
```

### Production (AWS)
```
EC2 (t3.small)     → Docker Compose (all services) + Nginx SSL
RDS MySQL          → portfolio database
RDS PostgreSQL     → mcp_farm database
ElastiCache Redis  → BullMQ queue
S3                 → Frontend assets, DB backups
```

---

## 16. Technology Stack

| Layer | Technologies |
|-------|-------------|
| Backend | Java 23, Spring Boot 3.2, Maven, JPA/Hibernate, MySQL 8 |
| Frontend | React 19, TypeScript 5.9, Vite 8, TailwindCSS 4, Zustand 5, React Query 5 |
| Infrastructure | Node.js 18, Fastify v4, Drizzle ORM, PostgreSQL 15 |
| AI | Anthropic Claude Sonnet 4.6, Vercel AI SDK v6 |
| MCP | @modelcontextprotocol/sdk v1.29, StreamableHTTP transport |
| Queue | BullMQ + Redis 7 |
| Broker | Zerodha Kite Connect (OAuth 2.0) |
| Charts | Recharts 3.8 |
| Real-time | STOMP over SockJS, SSE (Server-Sent Events) |
| Containers | Docker, Docker Compose, Nginx |
| API Docs | SpringDoc OpenAPI 3.0 + Swagger UI |

---

## 17. Design Principles

| Principle | Implementation |
|-----------|---------------|
| Apps own data, Agent Farm owns intelligence | Portfolio API manages domain data; Agent Farm orchestrates AI |
| User context via tool parameters | userId passed to MCP tools — language-agnostic |
| Defense in depth | 7 auth layers (JWT → API key → subscription → ownership) |
| Graceful degradation | Agent Farm down → Claude fallback. Kite down → CSV import |
| Full transparency | Every AI analysis saved with model, prompt, raw response |
| FIFO compliance | Tax engine uses legally required lot matching |
| Separation of concerns | Java for domain, Node.js for AI orchestration, React for UI |

---

## 18. Environment Variables

### Portfolio Tracker API
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DB_HOST` | Yes | `localhost` | MySQL host |
| `DB_PORT` | Yes | `3306` | MySQL port |
| `DB_NAME` | Yes | `portfolio` | Database name |
| `DB_USER` | Yes | `root` | MySQL user |
| `DB_PASSWORD` | Yes | — | MySQL password |
| `JWT_SECRET` | Yes | — | JWT signing key (min 32 chars) |
| `ANTHROPIC_API_KEY` | Yes | — | Claude API key |
| `AGENT_FARM_URL` | No | `http://localhost:8082` | Agent Farm URL |
| `AGENT_FARM_API_KEY` | No | `admin-secret` | Agent Farm auth |
| `KITE_API_KEY` | No | — | Zerodha API key |
| `KITE_API_SECRET` | No | — | Zerodha API secret |
| `FRONTEND_URL` | Yes | `http://localhost:5173` | CORS origin |

### MCP Farm (Gateway + Agent Farm)
| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | Yes | — | PostgreSQL connection |
| `REDIS_URL` | Yes | — | Redis connection |
| `ADMIN_API_KEY` | Yes | `admin-secret` | Admin authentication |
| `ANTHROPIC_API_KEY` | Yes | — | Claude API key |
| `OPENAI_API_KEY` | No | — | OpenAI fallback |
