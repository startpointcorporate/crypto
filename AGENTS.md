# AGENTS.md — Instructions for Codex

Repository: `startpointcorporate/crypto`  
Product: cross-exchange spot arbitrage, paper trading first.

## 1. Mission and delivery philosophy

Build a **small, testable, replayable** system that monitors spot order books on several exchanges, identifies potentially executable cross-exchange arbitrage, and evaluates the economics with virtual portfolios before any live trading.

Apply **Fail Fast, Learn Fast, Replay Fast**:
- Deliver the smallest useful vertical slice, runnable end to end.
- Every deployment, database migration, configuration, and test must be reproducible from version-controlled scripts/manifests.
- Every decision should be explainable and replayable from recorded inputs; do not depend on live APIs during replay.
- Prefer measurable experiments over speculative architecture. Remove unnecessary abstraction and infrastructure.
- Do not implement strategy discovery, LLM agents, TradingAgents or Freqtrade. Arbitrage calculations are deterministic.
- Do not claim profitability from displayed last prices; use executable bid/ask, size, fees, data freshness and rebalancing costs.

**Hard boundary:** the MVP is PAPER ONLY. Never place real orders, transfer funds, or request trading-enabled API credentials as part of the MVP. Production execution is a separate, explicitly approved phase.

## 2. Technology invariants (non-negotiable)

- **C# / .NET 11**: target `net11.0`; pin an appropriate .NET 11 SDK in `global.json` and build .NET 11 runtime images. Check actual available SDK/package versions before pinning; do not silently downgrade to .NET 10.
- **ASP.NET Core REST API** with OpenAPI, versioned routes, typed request/response contracts and validation. This API is the contract for a future **.NET MAUI** client and/or integration in the **Konektag website**; do not build a UI now.
- **OpenTelemetry** for traces, metrics, and structured logging. Export via OTLP to the **existing telemetry pipeline** so metrics/traces/logs can feed the existing Grafana/Prometheus/Loki stack as supported by its actual configuration. Do not install a second monitoring stack or assume Grafana is an OTLP receiver. Inspect actual collector/Alloy endpoints before deploying.
- Deploy to the **existing K3s** cluster using version-controlled, repeatable Kubernetes manifests or a small Helm chart; use readiness/liveness probes and resource requests/limits.
- Use the **shared PostgreSQL** service and a dedicated application database/schema and least-privilege role as appropriate to existing database conventions; do not provision a new PostgreSQL server or embed PostgreSQL in the application chart.
- Use **Kubernetes Secrets** for connection strings and (in future) exchange credentials. Secret *references* may be versioned; secret values must not be committed or logged. For deployment, use `SecretKeyRef`; do not rely on plaintext secrets in ConfigMaps, manifests or .env files committed to Git.
- Linux containers, UTC timestamps, decimal arithmetic for quantities, prices and fees (never binary floating point for monetary calculations).
- Use dependency injection, `HttpClientFactory`, cancellation tokens, timeouts, bounded retries with backoff, and explicit handling of exchange rate limits.
- Prefer mature, maintained .NET exchange SDKs where they demonstrably support the required endpoints; otherwise implement a narrow adapter. Verify capabilities (especially Bitpanda Fusion) rather than inventing endpoints.

## 3. Architecture: simple now, extensible later

Start with a **modular monolith** deployed as one application/process where feasible:
- REST API
- Background market-data collector / arbitrage scanner
- Paper execution and virtual portfolio ledger
- PostgreSQL persistence
- Telemetry

Use clear internal boundaries, not premature microservices:
1. **Domain**: instruments, venues, fee schedules, balances, opportunities, arbitrage trades, events.
2. **Exchange adapters**: normalized market data and, in a later phase, authenticated trading operations.
3. **Application services**: scanner, risk validation, execution orchestration, accounting, reporting.
4. **Infrastructure**: PostgreSQL, hosted workers, telemetry, Kubernetes config.
5. **REST API**: transport only; no arbitrage calculations in controllers/endpoints.

Define interfaces at actual boundaries (exchange data, execution, clock, persistence), not one interface per class. Use explicit, minimally coupled domain contracts so adding an exchange does not change the core algorithm.

An exchange may be configured/enabled/disabled independently. Support OKX Europe, Kraken and Bitpanda Fusion as **candidate** venues; start with the first **two proven to expose suitable public order-book data**. Do not assume Bitpanda Fusion's public depth endpoints are available without authentication. Coinbase Advanced may be added later. Verify symbol availability and local regulatory/product constraints; do not hardcode USDT availability in the EEA.

## 4. Phase 1 — MVP paper trading (only authorized execution mode)

Initial scope:
- 2 exchanges first, extend to 3 and then optionally 4 via adapters.
- 1 common, verified spot trading pair first (prefer BTC/EUR if supported across the selected venues), then additional common pairs only after proof of value.
- Virtual budget: **100 EUR equivalent per exchange**. Make allocations configurable, not hardcoded.
- Start with pre-funded paper inventories, e.g. 50 EUR quote currency and 50 EUR equivalent base asset per exchange; persist the exact initial conversion valuation and prices.
- Observe public best bid/ask and sufficient order-book depth. Normalize instrument and currencies, tick sizes, lot sizes, minimum notionals and data timestamps.
- Compute both arbitrage directions between each exchange pair. Never compare last-traded prices as executable offers.
- Estimate all-in net profitability at a **bounded executable quantity**, including buy/sell fees, depth-weighted execution price, slippage/latency assumptions, rounding, any conversion costs and an explicit rebalancing cost provision.
- Paper-execute both legs against observed book liquidity and venue-specific virtual balances. Prevent double spend; record partial/unfilled leg scenarios and exposure rather than assuming guaranteed atomic execution.
- Persist market-data evidence used for each decision (or a compact, lossless event record adequate for deterministic replay), trade intents, simulated orders/fills, fees, balance changes, performance and rejection reasons.
- Show **arbitrage PnL separately from mark-to-market changes in crypto inventory**, and track inventory drift, drawdown, net profit, executed/not executed opportunities and rebalancing costs.
- Use exchange timestamps and local receipt timestamps, impose maximum quote age and avoid combining stale/inconsistent books.
- No real API trading keys, real orders, real deposits, withdrawals or transfers in this phase.
- No user-facing UI in the MVP.

**Critical accounting rule:** capital is always attached to an exchange *and* an asset. Never model a single globally spendable balance across venues. Each leg must pass independent balance and venue constraints.

### Capital management: design for growth from day one

The product must support **adding capital to any exchange independently**, first as a virtual deposit, and later via reconciliation of actual external deposits:
- Model a per-exchange, per-asset ledger with explicit events: `InitialAllocation`, `Deposit`, `Withdrawal`, `TradeFill`, `Fee`, `Transfer`, `Adjustment` (only implement what MVP needs).
- Deposits/withdrawals and internal transfers are **not trading profit**. Performance reports must exclude external cash flows.
- REST API must allow adding virtual capital to an identified exchange/asset in paper mode, with amount, reference, timestamp and idempotency key; validate positive amounts and avoid duplicates.
- Keep an immutable audit trail for all funding/balance movements. Do not mutate starting capital history to represent subsequent deposits.
- Prepare explicit support for new exchanges, multiple assets, capital limits, future rebalancing and multi-venue reporting without creating a general-purpose financial platform.
- For live trading later, actual deposits normally occur at the venue; the system must **observe/reconcile** those deposits, not assume that a REST call to our API transfers money.

## 5. Phase 2 — production readiness (NOT part of initial implementation)

Separate paper and live execution behind an execution interface with a strict mode boundary:
- Default to `Paper`; use a **fail-closed** configuration. Enabling `Live` requires a separately reviewed deployment/configuration and explicit operational approval, not a toggle available through a public API.
- Add authenticated venue adapters and secure, least-privilege exchange credentials only when live trading is explicitly authorized.
- Pre-trade limits per venue/asset, max order notional, exposure limits, kill switch, stale-data guards, order idempotency, order reconciliation and crash recovery.
- Handle independent fills, partial fills, one-leg failure, locked balances and timeouts. Never claim cross-exchange atomicity.
- Verify balances and exchange order status after restart before placing any new live order.
- Provide a dedicated reconciliation process for external deposits, withdrawals, fees, and exchange balances.
- No withdrawal permissions for trading keys unless a separately approved feature explicitly requires them; prefer trade-only credentials.
- Test mode isolation so a paper run cannot accidentally call live order endpoints.
- Compliance and available assets/exchanges must be verified for the legal entity and country before funding production accounts.

Do not implement these production features prematurely; preserve the boundaries so they can be added safely.

## 6. REST API contract (illustrative v1)

Expose `/api/v1` endpoints with OpenAPI and machine-readable errors. Keep API decoupled from MAUI/Konektag:

- `GET /health/live` and `GET /health/ready`
- `GET /api/v1/exchanges` — configured venues and connectivity status
- `GET /api/v1/markets` — supported normalized pairs
- `GET /api/v1/opportunities` — detected opportunities, timestamps and net estimates
- `GET /api/v1/portfolios` — current balances by exchange/asset and total valuation
- `GET /api/v1/trades` — paper trade and fill history
- `GET /api/v1/performance` — PnL decomposition, fees, drawdown, invested capital
- `POST /api/v1/paper/deposits` — **virtual** capital addition with idempotency key
- `GET /api/v1/paper/ledger` — capital movements and accounting history

Implement only the endpoints needed by the current vertical slice. State-changing API operations require authentication/authorization (even in paper mode); do not leave a funding endpoint publicly writable. Use stable DTOs so client integration does not depend on internal persistence entities.

## 7. PostgreSQL and data integrity

- Use EF Core + Npgsql (or equivalently supported .NET 11-compatible drivers) with version-controlled migrations; select versions compatible with the chosen SDK.
- Database credentials come from K3s Secret references.
- Maintain unique idempotency constraints for deposits, simulated order intents and event processing.
- Use transactions for virtual ledger updates; enforce nonnegative available balances where applicable, and guard against concurrent spending.
- Do not log raw secrets or personally identifying credential data.
- Retain the minimum market observations needed for test reproducibility; configure retention/rollup before unlimited high-frequency persistence.

## 8. OpenTelemetry, reliability and K3s

- Instrument market-data lag, disconnected streams, quote age, opportunity rate, accepted/rejected simulations, estimated net spread, fill failures, portfolio valuation, portfolio PnL, process health, and API latency/errors.
- Label metrics with low-cardinality values only (exchange, pair, outcome); never use order IDs or timestamps as metric labels.
- Use structured logs and correlation/trace IDs through scanner, decision, execution and ledger.
- Export OTLP to the existing **Alloy / collector** endpoint (after discovery); verify routing into existing Prometheus/Grafana/Loki and traces if a tracing backend is actually present. Do not presume Tempo is installed.
- Kubernetes manifests/Helm: namespace parameterized, single replica initially, health probes, Secret references, configuration via ConfigMap/env, safe rolling deploys, resource limits, no public ingress by default.
- Do not bind to the shared PostgreSQL admin account; use a dedicated least-privilege user. Do not create cluster-wide resources without a demonstrated need.
- Initial resource budget to verify under load: **1 vCPU, 1 GiB memory** for application; tune from measurements.

## 9. Tests and replay (required, not optional)

- Unit tests for spread, fees, depth/VWAP, rounding, minimum order size, balance checks and profit accounting.
- Deterministic integration test with recorded books for two exchanges and a virtual 100-EUR balance per exchange.
- Tests for no-op when fees remove the apparent arbitrage, stale book, insufficient funds, partial leg, duplicated events, duplicate deposit request, restart/recovery and correct PnL when capital is added mid-test.
- Replay command or test harness must process a fixed snapshot/event fixture with a virtual clock and reproduce identical orders, ledger and PnL without contacting external APIs.
- Contract tests for adapters against documented payloads; live API smoke tests should be optional and read-only.
- Build/test/package/deploy instructions must be executable from a clean checkout.

## 10. Implementation order — thin vertical slices

1. Bootstrap .NET 11 solution, API, Worker, OpenAPI, health checks, Dockerfile, tests, PostgreSQL migrations, OTEL instrumentation and basic K3s deployment.
2. Implement **two** read-only public exchange adapters with timestamped order-book snapshots for one common pair; record reproducible fixtures.
3. Add fee/depth-aware opportunity calculation and explicit rejection reasons.
4. Add persistent virtual exchange inventories, simulated dual-leg execution and `POST /api/v1/paper/deposits`.
5. Expose portfolio, trades, performance; run paper trading and replay from saved observations.
6. Only then add the third venue, additional pairs or production preparation based on observed evidence.

Do not produce a large speculative backlog, premature multi-service architecture, dashboards, live-trading code or unnecessary abstractions. Prefer a working end-to-end MVP.

## 11. Codex working agreement

Before implementing a task:
1. Inspect existing code and these instructions; preserve established conventions.
2. Identify the smallest complete change and the acceptance criteria.
3. Implement and run the relevant tests.
4. Document exact commands to build, test, deploy and replay.
5. Report what actually ran, what was not verified and any assumption requiring live exchange/API confirmation.

Never claim a deployment, API test, trade, metric or exchange capability was verified if it was not. Never invent production URLs, secret names, database hosts, namespace names, market availability, fees or permissions: make them configurable and explicitly mark them for discovery.

**Success criterion for phase 1:** the system runs continuously on K3s, compares real public order-book data from at least two MiCA-appropriate spot venues, performs **paper-only** two-leg arbitrage with per-venue virtual capital, allows additional virtual deposits, produces net performance after realistic costs, and can replay the same decisions deterministically.
