# FinAlly — Build Plan

App root: `~/projects/portfolio/finally/`
Spec: `~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md`
Dispatch from: `~/projects/gascity/gastown/`

All `gc sling` commands are run from the gastown city root.

---

## Status

| Step | Description | Status |
|------|-------------|--------|
| 1 | Repo scaffold | ✅ done |
| 2 | Backend foundation | ✅ done |
| 3 | Market data — simulator + SSE | ✅ done |
| 4 | Portfolio API | ✅ done |
| 5 | Watchlist API | ✅ done |
| 6 | LLM chat integration | ✅ done |
| 7 | Frontend scaffold | ✅ done |
| 8 | Frontend: watchlist panel + SSE | ✅ done |
| 9 | Frontend: charts | ⬜ todo |
| 10 | Frontend: portfolio views | ⬜ todo |
| 11 | Frontend: trade bar | ⬜ todo |
| 12 | Frontend: AI chat panel | ⬜ todo |
| 13 | Dockerfile + docker-compose | ⬜ todo |
| 14 | Start/stop scripts | ⬜ todo |
| 15 | Backend unit tests | ⬜ todo |
| 16 | Frontend unit tests | ⬜ todo |
| 17 | E2E Playwright tests | ⬜ todo |
| 18 | Full integration smoke test | ⬜ todo |
| 19 | Move rig from portfolio/ to finally/ | ✅ done |

---

## Step 1 — Repo scaffold

Create the directory structure, git repo, .gitignore, and environment files.

```bash
gc sling claude "
Create the project scaffold for ~/projects/portfolio/finally/ per the spec at
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 4.

Tasks:
- mkdir -p finally/{frontend,backend/db,scripts,test,db}
- git init inside finally/
- Create .gitignore: node_modules/, __pycache__/, *.pyc, .env, db/finally.db, .next/, out/, .uv/
- Create db/.gitkeep
- Create .env.example with OPENROUTER_API_KEY=, MASSIVE_API_KEY=, LLM_MOCK=false
- Create a minimal README.md: project name, one-line description, 'docker run' quickstart
- git add and initial commit
Work dir: ~/projects/portfolio/
" --rig portfolio
```

Done when: `~/projects/portfolio/finally/` exists with git repo, .gitignore, .env.example, db/.gitkeep, README.md.

---

## Step 2 — Backend foundation

FastAPI app with uv, SQLite lazy init, all schema tables, seed data, health endpoint.

```bash
gc sling claude "
Set up the FastAPI backend for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md sections 6, 8, 9.

Tasks:
- uv init inside backend/ with Python 3.12
- uv add fastapi uvicorn[standard] aiosqlite python-dotenv
- Create main.py: FastAPI app, mounts static files from ../static/, includes all routers
- Create db/schema.py: lazy init function that creates all 6 tables (users_profile,
  watchlist, positions, trades, portfolio_snapshots, chat_messages) and seeds default data
  (user_id='default' cash=10000, 10 default tickers) if not already present
- Create api/health.py: GET /api/health returns {status: ok}
- Wire db init to FastAPI startup event
- Verify: uv run uvicorn main:app starts without error and GET /api/health returns 200
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `uv run uvicorn main:app` starts, `/api/health` returns 200, DB initializes with seed data.

---

## Step 3 — Market data: simulator + SSE

GBM price simulator, shared price cache, SSE streaming endpoint.

```bash
gc sling claude "
Implement market data for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 7.

Tasks:
- Create market/interface.py: abstract MarketDataProvider with get_prices() and start/stop
- Create market/simulator.py: GBM simulator
  - Seed prices: AAPL=190, GOOGL=175, MSFT=415, AMZN=185, TSLA=175, NVDA=875, META=505, JPM=200, V=275, NFLX=630
  - Updates every 500ms using GBM (drift=0.0001, vol=0.002 per tick, correlated moves)
  - Occasional random events: 2-5% spike on a random ticker every ~30s
  - Runs as asyncio background task
- Create market/cache.py: in-memory dict {ticker: {price, prev_price, timestamp, direction}}
- Create api/stream.py: GET /api/stream/prices SSE endpoint
  - Long-lived, pushes all cached prices every 500ms
  - Each event: {ticker, price, prev_price, timestamp, direction}
  - Uses FastAPI StreamingResponse with text/event-stream
- Wire simulator start/stop to FastAPI lifespan
- Verify: curl /api/stream/prices streams JSON events continuously
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `/api/stream/prices` streams price events continuously at ~500ms intervals.

---

## Step 4 — Portfolio API

Trade execution, positions tracking, P&L snapshots, portfolio summary.

```bash
gc sling claude "
Implement the portfolio API for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md sections 8, 9.

Tasks:
- Create api/portfolio.py with three endpoints:
  GET /api/portfolio
    - Returns: {cash_balance, total_value, positions: [{ticker, quantity, avg_cost,
      current_price, unrealized_pnl, pnl_pct}], total_unrealized_pnl}
    - current_price from price cache; unrealized_pnl = (current - avg_cost) * quantity
  POST /api/portfolio/trade {ticker, quantity, side}
    - Validate: sufficient cash for buy; sufficient shares for sell
    - Execute: update positions (avg cost for buys, FIFO not needed), update cash, insert trade row
    - Immediately record a portfolio snapshot after execution
    - Return: updated portfolio summary + trade confirmation
  GET /api/portfolio/history
    - Returns portfolio_snapshots ordered by recorded_at for P&L chart
- Create background task: record portfolio snapshot every 30s
- Wire snapshot task to FastAPI lifespan
- Verify: buy AAPL 10 shares, check cash decreased, position appears, history has entry
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: trade round-trip works, cash updates correctly, snapshots recorded.

---

## Step 5 — Watchlist API

CRUD for the user's watchlist, prices included in responses.

```bash
gc sling claude "
Implement the watchlist API for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 9.

Tasks:
- Create api/watchlist.py with three endpoints:
  GET /api/watchlist
    - Returns [{ticker, price, prev_price, direction, added_at}] from watchlist table
      enriched with current prices from cache
  POST /api/watchlist {ticker}
    - Validate: ticker is non-empty string (uppercase it)
    - Insert into watchlist (ignore duplicate via UNIQUE constraint)
    - Return updated watchlist
  DELETE /api/watchlist/{ticker}
    - Remove from watchlist
    - Return updated watchlist
- Verify: add PYPL, confirm it appears; delete PYPL, confirm it's gone
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: all three watchlist endpoints return correct data.

---

## Step 6 — LLM chat integration

OpenRouter/LiteLLM chat endpoint with structured output, auto-trade execution.

```bash
gc sling claude "
Implement the LLM chat endpoint for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 10.

Tasks:
- uv add litellm
- Create api/chat.py: POST /api/chat {message}
  1. Load portfolio context (cash, positions+P&L, watchlist+prices, total value)
  2. Load last 20 chat_messages from DB
  3. Build system prompt: FinAlly persona, portfolio context injected
  4. Call LiteLLM: model='openrouter/openai/gpt-oss-120b', api_key=OPENROUTER_API_KEY,
     structured output matching schema {message, trades?, watchlist_changes?}
  5. Auto-execute trades (same validation as /api/portfolio/trade)
  6. Auto-execute watchlist changes
  7. Store user message and assistant response in chat_messages
  8. Return {message, trades_executed, watchlist_changes_executed, errors}
- Create mock mode: if LLM_MOCK=true, return deterministic response
  {message: 'Mock response', trades: [], watchlist_changes: []}
- Verify with LLM_MOCK=true: POST /api/chat returns valid JSON
- Verify with real key: ask 'What is my portfolio worth?' gets a real response
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `/api/chat` returns structured JSON with mock mode; real mode calls OpenRouter.

---

## Step 7 — Frontend scaffold

Next.js 14 with TypeScript, Tailwind, dark theme config, static export, proxy to backend in dev.

```bash
gc sling claude "
Scaffold the Next.js frontend for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md sections 3, 11.

Tasks:
- npx create-next-app@latest frontend --typescript --tailwind --app --no-src-dir --import-alias '@/*'
- Configure next.config.ts: output: 'export', trailingSlash: true
- Configure Tailwind: extend theme with custom colors
    background: '#0d1117', surface: '#1a1a2e'
    accent: '#ecad0a', primary: '#209dd7', secondary: '#753991'
- Create globals.css: dark body background, base font (monospace stack suits terminal feel)
- Create layout.tsx: dark html/body, viewport meta, title 'FinAlly'
- Create a placeholder page.tsx: single div 'FinAlly loading...' in terminal style
- Verify: npm run build produces static export in out/; npm run dev serves on :3000
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `npm run build` succeeds, static export in `out/`, dark theme applied.

---

## Step 8 — Frontend: watchlist panel + SSE connection

SSE hook, price state, watchlist grid with flash animations.

```bash
gc sling claude "
Build the watchlist panel and SSE connection for ~/projects/portfolio/finally/frontend/
per ~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md sections 2, 11.

Tasks:
- Create hooks/usePrices.ts: EventSource hook connecting to /api/stream/prices
  - Maintains {[ticker]: {price, prev_price, direction, timestamp}} state
  - Auto-reconnects on error (EventSource does this natively)
  - Exposes connectionStatus: 'connected' | 'reconnecting' | 'disconnected'
- Create hooks/useWatchlist.ts: fetches GET /api/watchlist, exposes add/remove functions
- Create components/WatchlistPanel.tsx:
  - Grid of ticker rows: symbol, price (2dp), change % (colored), sparkline placeholder
  - Price flash: on price update, apply bg-green-flash or bg-red-flash CSS class for 500ms
  - Click row to select ticker (lifted state)
  - Add ticker input + button at bottom
- Create components/ConnectionStatus.tsx: colored dot in header (green/yellow/red)
- Wire into page.tsx with basic two-column layout
- Price flash CSS: define keyframe animations in globals.css
- Verify: open page, prices stream and flash, add/remove ticker works
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: prices stream to the watchlist panel, flash animations visible, add/remove works.

---

## Step 9 — Frontend: charts

Main ticker chart and sparkline mini-charts using Lightweight Charts.

```bash
gc sling claude "
Build the chart components for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 11.

Tasks:
- npm install lightweight-charts
- Create components/Sparkline.tsx:
  - Accepts priceHistory: number[] (accumulated from SSE since page load)
  - Renders a tiny area chart (no axes, no labels) using Lightweight Charts
  - Green if latest > first, red if latest < first
- Update WatchlistPanel to pass accumulated price history per ticker to Sparkline
  (store price history in usePrices hook, capped at last 100 points)
- Create components/MainChart.tsx:
  - Accepts selectedTicker and its full price history
  - Renders a larger line/area chart with time axis and price axis
  - Updates in real time as new SSE prices arrive
  - Shows ticker name and current price in chart header
- Place MainChart in the main content area of page.tsx
- Verify: select a ticker, main chart shows and updates live; sparklines fill in over time
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: main chart updates live, sparklines fill progressively from SSE data.

---

## Step 10 — Frontend: portfolio views

Positions table, portfolio heatmap (treemap), P&L chart.

```bash
gc sling claude "
Build the portfolio view components for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 11.

Tasks:
- npm install recharts (for treemap and P&L line chart)
- Create hooks/usePortfolio.ts: fetches GET /api/portfolio every 5s and after trades
- Create hooks/usePortfolioHistory.ts: fetches GET /api/portfolio/history every 30s
- Create components/PositionsTable.tsx:
  - Columns: Ticker | Qty | Avg Cost | Price | Unr. P&L | P&L %
  - P&L colored green (positive) or red (negative)
  - Empty state: 'No positions yet'
- Create components/PortfolioHeatmap.tsx:
  - Recharts Treemap: each cell is a position, sized by portfolio weight (value/total)
  - Cell color: green gradient for positive P&L, red gradient for negative
  - Cell label: ticker + P&L %
- Create components/PnLChart.tsx:
  - Recharts LineChart of portfolio_snapshots.total_value over time
  - X-axis: time, Y-axis: dollar value
  - Tooltip shows value and timestamp
- Arrange in page.tsx: heatmap + P&L chart in a panel below the main chart
- Verify: positions appear after a trade, heatmap reflects weights, P&L chart has data points
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: all three portfolio components render with live data after executing a trade.

---

## Step 11 — Frontend: trade bar

Ticker + quantity input, buy/sell buttons, instant feedback.

```bash
gc sling claude "
Build the trade bar for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 11.

Tasks:
- Create components/TradeBar.tsx:
  - Inputs: ticker (text, auto-uppercase), quantity (number, min=0.01)
  - Buttons: BUY (primary blue) and SELL (red)
  - On submit: POST /api/portfolio/trade, show brief success/error inline message
  - Clears quantity after successful trade; keeps ticker for follow-on trades
  - Disabled state while request in flight
  - Pre-fills ticker when user clicks a watchlist row (lifted selectedTicker state)
- Trigger portfolio refresh (usePortfolio) after successful trade
- Place TradeBar in the header or below the watchlist panel
- Verify: buy 5 AAPL, see cash decrease and position appear without page refresh
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: buy/sell round-trip updates portfolio state without page reload.

---

## Step 12 — Frontend: AI chat panel

Collapsible chat sidebar, message history, inline trade confirmations.

```bash
gc sling claude "
Build the AI chat panel for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md sections 2, 11.

Tasks:
- Create components/ChatPanel.tsx:
  - Fixed right sidebar, collapsible via toggle button
  - Scrollable message history: user messages right-aligned, assistant left-aligned
  - Text input + send button at bottom; submit on Enter
  - Loading indicator (spinner or typing dots) while awaiting response
  - On response: render message text; if trades_executed or watchlist_changes_executed,
    show inline confirmation chips below the message
    e.g. 'Bought 5 AAPL @ $190.42' in green, 'Added PYPL to watchlist' in blue
  - Error display if API call fails
- POST /api/chat on send, append both user and assistant messages to local state
- Trigger portfolio + watchlist refresh after any executed actions
- Wire ChatPanel into page.tsx layout (right sidebar)
- Verify with LLM_MOCK=true: send message, get mock response, panel updates correctly
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: chat panel sends/receives messages, inline action confirmations appear.

---

## Step 13 — Dockerfile + docker-compose

Multi-stage build, volume mount, single port 8000.

```bash
gc sling claude "
Create the Dockerfile and docker-compose.yml for ~/projects/portfolio/finally/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 12.

Tasks:
- Create Dockerfile (multi-stage):
  Stage 1 (node:20-slim): cd frontend && npm ci && npm run build
  Stage 2 (python:3.12-slim):
    - Install uv via pip
    - COPY backend/ /app/backend/
    - COPY --from=0 /app/frontend/out/ /app/static/
    - WORKDIR /app/backend
    - RUN uv sync --frozen
    - EXPOSE 8000
    - CMD uv run uvicorn main:app --host 0.0.0.0 --port 8000
  Ensure FastAPI serves /app/static/ at /* and /app/db/ is the DB volume path
- Create docker-compose.yml:
    services.app: build ., ports 8000:8000, volumes finally-data:/app/db,
    env_file .env
- Create .env.example (if not already present)
- Verify: docker compose up --build serves the full app on localhost:8000
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `docker compose up --build` serves the complete app on port 8000.

---

## Step 14 — Start/stop scripts

Idempotent shell scripts for macOS/Linux and Windows.

```bash
gc sling claude "
Create start/stop scripts for ~/projects/portfolio/finally/scripts/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 12.

Tasks:
- scripts/start_mac.sh:
  - Checks if container 'finally' is already running; skips start if so
  - docker build if image missing or --build flag passed
  - docker run -d --name finally -v finally-data:/app/db -p 8000:8000 --env-file ../.env finally
  - Waits up to 10s for /api/health to return 200
  - Prints: 'FinAlly running at http://localhost:8000'
  - Optionally: open http://localhost:8000 (macOS: open, Linux: xdg-open)
- scripts/stop_mac.sh:
  - docker stop finally && docker rm finally
  - Does NOT remove the volume
  - Prints: 'FinAlly stopped. Data preserved in finally-data volume.'
- scripts/start_windows.ps1 and stop_windows.ps1: PowerShell equivalents
- chmod +x scripts/*.sh
- Verify: ./scripts/start_mac.sh starts the app; ./scripts/stop_mac.sh stops it cleanly
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: start/stop scripts work idempotently on macOS.

---

## Step 15 — Backend unit tests

pytest coverage for simulator, portfolio logic, and API routes.

```bash
gc sling claude "
Write backend unit tests for ~/projects/portfolio/finally/backend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 13.

Tasks:
- uv add --dev pytest pytest-asyncio httpx
- tests/test_simulator.py:
  - GBM produces prices within ±30% of seed after 1000 ticks
  - Direction field correct (up/down/flat)
  - All 10 default tickers present
- tests/test_portfolio.py:
  - Buy: cash decreases by quantity*price, position created with correct avg_cost
  - Buy more: avg_cost updates correctly
  - Sell partial: quantity decreases, cash increases
  - Sell all: position row removed
  - Sell more than owned: raises validation error
  - Buy with insufficient cash: raises validation error
  - P&L calculation: unrealized_pnl = (current - avg_cost) * quantity
- tests/test_api.py (using httpx TestClient):
  - GET /api/health → 200 {status: ok}
  - GET /api/watchlist → 200, contains 10 default tickers
  - POST /api/watchlist {ticker: PYPL} → 200, PYPL in list
  - DELETE /api/watchlist/PYPL → 200, PYPL gone
  - POST /api/portfolio/trade buy → 200, cash decreased
  - GET /api/portfolio → 200, correct schema
  - POST /api/chat with LLM_MOCK=true → 200, valid JSON
- Verify: uv run pytest passes all tests
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `uv run pytest` passes all backend tests.

---

## Step 16 — Frontend unit tests

React Testing Library tests for key components.

```bash
gc sling claude "
Write frontend unit tests for ~/projects/portfolio/finally/frontend/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 13.

Tasks:
- npm install --save-dev @testing-library/react @testing-library/jest-dom jest jest-environment-jsdom
- Configure jest for Next.js (jest.config.ts)
- tests/WatchlistPanel.test.tsx:
  - Renders ticker rows with prices
  - Price flash class applied on price change, removed after 500ms
  - Add ticker input submits and clears
- tests/PositionsTable.test.tsx:
  - Renders positions with correct P&L color (green/red)
  - Empty state shows message
- tests/ChatPanel.test.tsx:
  - Sends message on Enter key
  - Loading state shown while awaiting response
  - Mock response renders correctly
  - Trade confirmation chips appear when trades_executed is non-empty
- Verify: npm test passes all tests
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: `npm test` passes all frontend tests.

---

## Step 17 — E2E Playwright tests

Full stack tests in Docker with LLM_MOCK=true.

```bash
gc sling claude "
Write Playwright E2E tests for ~/projects/portfolio/finally/test/ per
~/projects/gascity/gastown/planning/PORTFOLIO_SPEC.md section 13.

Tasks:
- Create test/docker-compose.test.yml:
  - app service: finally image, LLM_MOCK=true, fresh volume
  - playwright service: mcr.microsoft.com/playwright, runs tests against app
- npm init inside test/, install @playwright/test
- test/e2e.spec.ts covering:
  1. Fresh start: 10 tickers visible, $10,000 balance shown, prices streaming
  2. Add ticker: type PYPL, click Add, verify PYPL appears
  3. Remove ticker: click remove on PYPL, verify gone
  4. Buy shares: enter AAPL 5, click BUY, verify cash decreases, position appears
  5. Sell shares: enter AAPL 2, click SELL, verify cash increases, position quantity updates
  6. Portfolio heatmap: at least one cell visible after buying
  7. P&L chart: has at least one data point
  8. Chat (mocked): type 'hello', send, verify response appears, no error
  9. SSE resilience: wait 2s for price updates to arrive
- Verify: docker compose -f test/docker-compose.test.yml up --exit-code-from playwright
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: all E2E scenarios pass in Docker.

---

## Step 18 — Full integration smoke test

Manual verification of the complete golden path.

```bash
gc sling claude "
Run a full integration smoke test for ~/projects/portfolio/finally/ and report results.

Steps to verify:
1. ./scripts/start_mac.sh --build (clean build from scratch)
2. Open http://localhost:8000 — confirm terminal UI loads, prices streaming
3. Click TSLA in watchlist — confirm main chart updates
4. Buy 10 TSLA — confirm cash decreases, position appears in table and heatmap
5. Sell 5 TSLA — confirm cash increases, position quantity halves
6. Add PYPL to watchlist — confirm it appears with live price
7. Remove PYPL — confirm it disappears
8. Chat: 'What is my portfolio worth?' — confirm real LLM response (needs OPENROUTER_API_KEY)
9. Chat: 'Buy 5 AAPL for me' — confirm trade executes, confirmation chip appears
10. ./scripts/stop_mac.sh — confirm container stops cleanly
11. ./scripts/start_mac.sh — confirm data persists (position still there)

Report any failures with exact error messages.
Work dir: ~/projects/portfolio/finally/
" --rig portfolio
```

Done when: all 11 smoke test steps pass, data persists across container restart.

---

## Step 19 — Move rig from portfolio/ to finally/

Re-register the rig so its root matches the app git repo.

```bash
gc rig remove portfolio
gc rig add ~/projects/portfolio/finally --name portfolio
```

Then update `city.toml` to reflect the new rig path (if `gc rig add` doesn't auto-update it):

```toml
[[rigs]]
name = "portfolio"

[rigs.imports.gastown]
source = "./assets/gastown"
```

Restart the city to pick up the new rig root:

```bash
gc stop && gc start
gc status   # portfolio/gastown.witness should come back up
gc doctor   # all green
```

Done when: `gc status` shows `portfolio/gastown.witness` running with rig root at `~/projects/portfolio/finally/`.

---

## Notes

- Steps 2–6 (backend) can be dispatched sequentially; each depends on the previous.
- Steps 7–12 (frontend) can begin once Step 2 is done (backend health endpoint needed for dev).
- Steps 8–12 can be dispatched in parallel once Step 7 (scaffold) is done.
- Steps 13–14 (Docker + scripts) depend on Steps 2–12 all being complete.
- Steps 15–16 (unit tests) can run in parallel with Steps 13–14.
- Step 17 (E2E) depends on Step 13.
- Step 18 depends on all prior steps.
- Step 19 can be done any time after Step 18 — it's a city config change, not app code.
