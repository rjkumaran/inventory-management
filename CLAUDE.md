# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Factory Inventory Management System Demo with GitHub integration - Full-stack application with Vue 3 frontend, Python FastAPI backend, and in-memory mock data (no database).

## Critical Tool Usage Rules

### Subagents
Use the Agent tool with these specialized subagents for appropriate tasks:

- **vue-expert**: Use for Vue 3 frontend features, UI components, styling, and client-side functionality
  - Examples: Creating components, fixing reactivity issues, performance optimization, complex state management
  - **MANDATORY RULE: ANY time you need to create or significantly modify a .vue file, you MUST delegate to vue-expert**
- **code-reviewer**: Use after writing significant code to review quality and best practices
- **Explore**: Use for understanding codebase structure, searching for patterns, or answering questions about how components work
- **general-purpose**: Use for complex multi-step tasks or when other agents don't fit

### Skills
- **backend-api-test** skill: Use when writing or modifying tests in `tests/backend` directory with pytest and FastAPI TestClient

### MCP Tools
- **ALWAYS use GitHub MCP tools** (`mcp__github__*`) for ALL GitHub operations
  - Exception: Local branches only - use `git checkout -b` instead of `mcp__github__create_branch`
- **ALWAYS use Playwright MCP tools** (`mcp__playwright__*`) for browser testing
  - Test against: `http://localhost:3000` (frontend), `http://localhost:8001` (API)

## Stack
- **Frontend**: Vue 3 + Composition API + Vite (port 3000)
- **Backend**: Python FastAPI (port 8001)
- **Data**: JSON files in `server/data/` loaded via `server/mock_data.py`

## Quick Start

```bash
# Backend (first run: uv venv && uv sync)
cd server
uv run python main.py        # http://localhost:8001 — API docs at /docs

# Frontend
cd client
npm install && npm run dev   # http://localhost:3000
npm run build                # Production build → client/dist/
```

macOS/Linux can use `./scripts/start.sh` and `./scripts/stop.sh`. On Windows, run each server manually in separate terminals.

No linters or formatters are configured (no ESLint, Prettier, Black, Ruff). Don't try to run lint commands — match the surrounding code style instead.

## Custom Commands

These slash commands are available in Claude Code:

- `/start` — Start both frontend and backend servers
- `/stop` — Stop both servers
- `/test` — Run frontend and backend tests with reporting
- `/demo-branch` — Create a new demo branch with auto-incrementing number
- `/reset-branch` — Switch to main and delete previous branch
- `/optimize` — Optimize the codebase

## Testing

The test suite lives at the repo root in `tests/` (not under `server/`). `pytest.ini` is in `tests/`, so commands must be run from there.

```bash
cd tests
uv run pytest -v                                    # All tests
uv run pytest backend/test_inventory.py -v          # Single file
uv run pytest backend/test_inventory.py::test_get_inventory -v   # Single test
uv run pytest -k "filter" -v                        # By name pattern
uv run pytest --cov=../server --cov-report=html     # With coverage
```

Tests use FastAPI `TestClient` (sync, no async needed). Shared fixtures live in `tests/backend/conftest.py`: `client`, `sample_inventory_item`, `sample_order`. There is no frontend test suite.

## Architecture

**Backend is a single-file API.** All endpoints live in `server/main.py`. Data is loaded once at startup from `server/data/*.json` via `server/mock_data.py` into module-level Python lists, then filtered in-memory per request. Mutations don't persist — restart the server to reset state. Pydantic models for request/response shapes are inline in `main.py`.

**Frontend is route-driven and filter-driven.** `client/src/main.js` registers routes for each view in `client/src/views/`. Every view loads raw data from `client/src/api.js` into `ref()` containers and exposes derived data through `computed()`. Cross-view shared state (filter selections, auth, i18n) lives in `client/src/composables/`.

**Filter System**: 4 filters (Time Period, Warehouse, Category, Order Status) apply to all data via query params. Time Period supports month names ("January") and quarters ("Q1").

**Data Flow**: Vue filter refs → `client/src/api.js` (axios + URLSearchParams) → FastAPI → in-memory filtering (`apply_filters()` / `filter_by_month()` in `main.py`) → Pydantic validation → Vue computed properties → template

**Reactivity**: Raw data in refs (`allOrders`, `inventoryItems`), derived data in computed properties — never recompute in methods.

**i18n**: `client/src/composables/useI18n.js` + `client/src/locales/en.js` and `ja.js`. Use the `t()` helper for all user-visible strings. Language switcher is in `App.vue` header.

## API Endpoints
- `GET /api/inventory` - Filters: warehouse, category
- `GET /api/orders` - Filters: warehouse, category, status, month
- `GET /api/dashboard/summary` - All filters
- `GET /api/demand`, `/api/backlog` - No filters
- `GET /api/spending/*` - Summary, monthly, categories, transactions
- `GET/POST/DELETE /api/tasks` - Task management
- `POST /api/purchase-orders` - Create purchase order from backlog item

## Code Conventions
- Always document non-obvious logic changes with comments

## Common Issues
1. Use unique keys in v-for (not `index`) — use `sku`, `month`, etc.
2. Validate dates before `.getMonth()` calls
3. Update Pydantic models when changing JSON data structure
4. Inventory filters don't support month (no time dimension)
5. Revenue goals: $800K/month single, $9.6M YTD all months

## File Locations
- Views: `client/src/views/*.vue`
- Components: `client/src/components/*.vue`
- Composables: `client/src/composables/` (useAuth, useFilters, useI18n)
- API Client: `client/src/api.js`
- Backend: `server/main.py`, `server/mock_data.py`
- Data: `server/data/*.json`
- Styles: `client/src/App.vue` (global CSS + design tokens)
- Tests: `tests/backend/`

## Design System
- Colors: Slate/gray (#0f172a, #64748b, #e2e8f0)
- Status: green/blue/yellow/red
- Charts: Custom SVG, CSS Grid for layouts
- No emojis in UI

## Sub-directory Guidance
- `client/CLAUDE.md` — Vue 3 patterns, reactivity pitfalls, component communication, i18n usage
- `server/CLAUDE.md` — FastAPI patterns, Pydantic models, filtering logic, error handling
- `tests/README.md` — Test suite structure, fixtures, coverage guidance
