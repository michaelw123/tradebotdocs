# Wheel + Defined-Risk Implementation Guide

This document is a technical and business implementation guide for the two options strategies in this repository:

- `WheelStrategy`
- `DefinedRiskTrendStrategy`

It consolidates architecture, design, workflow, data flow, signal generation, option selection, execution path, and backtest behavior.

## 1. Scope and Audience

Audience:
- Engineers maintaining strategy logic, execution, and backtest code.
- Strategy owners reviewing business rules and parameter behavior.
- Operators running paper/live sessions and reconciling results.

Out of scope:
- Broker account setup instructions.
- Tax/legal interpretation.
- Portfolio optimization beyond the current implemented rules.

## 2. High-Level System Architecture

Core execution model: Akka typed actors.

Main entry and wiring:
- `src/main/scala/tradebot/TradeBotMain.scala`
- `src/main/scala/tradebot/actors/TradeBotActor.scala`

Primary actor domains:
- Market data and indicator pipeline:
  - `src/main/scala/tradebot/actors/EquityMarketDataActor.scala`
  - `src/main/scala/tradebot/dto/EquityIndicatorBuilder.scala`
- Strategy actors:
  - `src/main/scala/tradebot/actors/strategy/WheelStrategyActor.scala`
  - `src/main/scala/tradebot/actors/strategy/DefinedRiskTrendStrategyActor.scala`
- Order orchestration:
  - `src/main/scala/tradebot/actors/OrderActor.scala`
  - `src/main/scala/tradebot/actors/order/wheel/WheelOrderService.scala`
  - `src/main/scala/tradebot/actors/order/definedrisk/DefinedRiskOrderService.scala`
- Broker gateway:
  - `src/main/scala/tradebot/actors/IBKRGatewayActor.scala`
  - `src/main/scala/tradebot/actors/IBKRGatewayProtocol.scala`
  - `src/main/scala/tradebot/actors/IBKRGatewayFlow.scala`
- Shared state and positions:
  - `src/main/scala/tradebot/actors/StateActor.scala`
- Persistence:
  - `src/main/scala/tradebot/actors/PersistenceActor.scala`
  - `src/main/scala/tradebot/persistence/SlickPersistenceAdapter.scala`

Option selection support (Wheel):
- `src/main/scala/tradebot/actors/wheel/WheelSecDefPlanner.scala`
- `src/main/scala/tradebot/actors/wheel/WheelQuoteScanMachine.scala`
- `src/main/scala/tradebot/actors/wheel/WheelOptionSelection.scala`
- `src/main/scala/tradebot/actors/wheel/WheelOptionBook.scala`

## 3. Configuration Architecture

Configuration source:
- `src/main/resources/application.conf`

Typed loader and validation:
- `src/main/scala/tradebot/utils/AppConfig.scala`

Key configuration objects:
- `AppConfig.WheelConfig`
- `AppConfig.DefinedRiskConfig`
- `AppConfig.WheelSymbolOverride`
- `AppConfig.DefinedRiskSymbolOverride`
- `AppConfig.IndicatorConfig`

Utility access and strategy-aware HTF mapping:
- `src/main/scala/tradebot/utils/ConfigUtils.scala`

Important design rule:
- Each symbol must belong to exactly one strategy block in `tradebot.equities`.
- Duplicate symbol assignment across strategy blocks fails validation at startup.

## 4. End-to-End Live Data and Control Flow

## 4.1 Session control

`TradeBotActor`:
- Wires all actors.
- Starts connectivity and subscriptions.
- Schedules reconnect.
- Schedules daily trading enable/disable windows.

## 4.2 Market data to indicators

`IBKRGatewayActor` forwards bars/ticks to `EquityMarketDataActor`.

`EquityMarketDataActor`:
- Aggregates incoming bars to LTF.
- Aggregates LTF to HTF (strategy-dependent HTF minutes).
- Updates indicators and regime in `Entity`.
- Dispatches `NewBar` to strategy actor bound to that symbol.

## 4.3 Strategy decision to order request

Strategy actor:
- Evaluates position context + indicators + filters.
- Emits order intent command to `OrderActor`.

`OrderActor`:
- Allocates order id.
- Performs buying-power and guard checks.
- Sends broker-side request via `IBKRGatewayActor`.

Gateway:
- Resolves contracts (especially options and combos).
- Sends final order to IBKR.

State and persistence:
- `StateActor` tracks positions/options/marks/IV rank.
- `PersistenceActor` records orders, legs, transactions.

## 5. Indicator and Signal Layer (Business Logic Foundation)

## 5.1 Regime and trend inputs

Built from `EquityIndicatorBuilder` outputs:
- HTF EMA fast/mid/slow.
- HTF ADX and trend thresholds (`equityTrendEnter`, `equityTrendExit`).
- HTF/LTF MACD.
- LTF RSI.
- Bollinger bands.
- VWAP deviation.
- MFI and ATR-percentile filters (configurable).

## 5.2 LTF signal generation

`LtfSignalDetector` classifies:
- `LongPullback`
- `ShortPullback`
- `LongReversion`
- `ShortReversion`
- `NoSignal`

Signal logic uses:
- Regime context (`Bullish`, `Bearish`, `RangeBound`, `Neutral`).
- RSI + Bollinger zone checks.
- MACD histogram turning behavior.
- ADX threshold for reversion.
- Optional MFI / ATR percentile filters.

File:
- `src/main/scala/tradebot/algos/LtfSignalDetector.scala`

## 6. Wheel Strategy Implementation

## 6.1 Business intent

Wheel cycle:
- Sell CSP when no stock.
- If assigned, hold shares.
- Sell CC against shares.
- Repeat.

Additional behaviors:
- Defensive roll near expiry or ITM pressure.
- Optional income roll.
- Profit-take close before expiry.

## 6.2 Entry gating

`WheelStrategyActor` + `wheel.decideWheelAction(...)` enforce:
- Position consistency guards.
- Regime gating:
  - CSP blocked in `Bearish`.
  - CC blocked in `Bullish`.
- Optional EMA trend filters.
- Optional LTF requirements.
- Optional VWAP deviation band.
- Optional IV-rank band.

Files:
- `src/main/scala/tradebot/actors/strategy/WheelStrategyActor.scala`
- `src/main/scala/tradebot/algos/wheel.scala`

## 6.3 Wheel option selection process

Pipeline:
1. Strategy emits open/roll request to `OrderActor`.
2. `OrderActor` sends `RequestWheelOption` or `RequestWheelRoll` to gateway.
3. Gateway resolves underlying conId and sec-def option params.
4. `WheelSecDefPlanner` picks expiry window and strike set.
5. `WheelQuoteScanMachine` requests batched quotes and accumulates bid/ask/last/close/mark.
6. Candidate diagnostics are computed per strike:
   - spread abs/pct
   - model delta
   - annualized credit
   - weighted score
   - reject reason
7. Best candidate is selected.
8. Gateway sends `PlaceResolvedWheelOption` or `PlaceResolvedWheelRoll` to `OrderActor`.
9. `WheelOrderService` performs final buy-power/market-guard checks and submits order.

Option scoring/selection files:
- `src/main/scala/tradebot/actors/wheel/WheelSecDefPlanner.scala`
- `src/main/scala/tradebot/actors/wheel/WheelQuoteScanMachine.scala`
- `src/main/scala/tradebot/actors/wheel/WheelOptionSelection.scala`

## 6.4 Execution design notes

Current behavior:
- Wheel open/roll orders are submitted as `MKT` (market), with guardrails on spreads and time window.
- Resolved limit estimate is logged and passed around but not used as active limit order type for live submission.

Files:
- `src/main/scala/tradebot/actors/order/wheel/WheelOrderService.scala`

Implication:
- Selection quality can still diverge from fill quality when live spreads move.

## 6.5 Wheel state tracking and pending controls

`WheelStrategyActor` maintains:
- `pendingOpenBySymbol`
- `pendingRollBySymbol`

Behavior:
- Blocks new wheel actions for symbol while pending.
- Pending clears on:
  - Position snapshot confirmation.
  - Explicit release from `OrderActor` on cancel/error statuses.

`OrderActor` release statuses:
- `Cancelled`
- `ApiCancelled`
- `Inactive`

Files:
- `src/main/scala/tradebot/actors/strategy/WheelStrategyActor.scala`
- `src/main/scala/tradebot/actors/OrderActor.scala`

## 7. Defined-Risk Trend Strategy Implementation

## 7.1 Business intent

Strategy family:
- Bull trend: short put vertical (`DefinedRiskBullPut`).
- Bear trend: short call vertical (`DefinedRiskBearCall`).
- Optional range-bound mode: iron condor.

Core goals:
- Directional premium capture with capped theoretical loss via spread width.
- Controlled entry frequency through cooldown and position state.

## 7.2 Entry logic

`DefinedRiskTrendStrategyActor`:
- Computes trend checks from HTF EMA stack and regime.
- Applies optional VWAP deviation filters.
- Applies cooldown bars.
- Blocks new entries if short side already present or pending close.

Iron condor entry:
- Requires `RangeBound`.
- Requires condor-enabled.
- Requires time window.
- Requires range-bound streak.
- Requires abs VWAP and ADX bounds.

File:
- `src/main/scala/tradebot/actors/strategy/DefinedRiskTrendStrategyActor.scala`

## 7.3 Exit logic

Vertical and condor exits include:
- Expiry.
- Profit target (mark improvement).
- Stop condition (mark deterioration).
- Opposite signal/regime-driven close.

Actor uses `pendingCloseBySymbol` to avoid duplicate close commands until state confirms resolution.

## 7.4 Contract construction and order routing

`DefinedRiskOrderService`:
- Builds target expiries/strikes from moneyness and spread width.
- Builds combo contracts (vertical or condor).
- Persists combo legs.
- Sends broker combo request / final order.

Current execution:
- Defined-risk combos submitted as `MKT`.

Files:
- `src/main/scala/tradebot/actors/order/definedrisk/DefinedRiskOrderService.scala`
- `src/main/scala/tradebot/actors/definedrisk/DefinedRiskConIdBook.scala`

## 8. Data Model and State Flow

Key runtime DTO/state objects:
- `Entity`: per-symbol rolling indicator state.
- `MarketData`: map of entities + market context.
- `SharedState`: account, positions, option positions, marks, IV rank snapshots.
- `WheelPositionSnapshot`: long stock shares + short put/call contracts.

State ownership:
- `EquityMarketDataActor` owns indicator progression.
- `StateActor` owns account/position/option mark/IV rank state.
- Strategy actors are consumers of updates and emit order intents.

## 9. Backtest Architecture and Workflow

Main entry:
- `src/main/scala/tradebot/utils/BacktestRunner.scala`

Capabilities:
- Symbol selection, date filters, train/test walk-forward.
- Extensive CLI overrides for wheel and defined-risk parameters.
- Fee/slippage support.
- Strategy-specific simulation paths.

Wheel backtest engine:
- `src/main/scala/tradebot/utils/BacktestWheelLifecycle.scala`
- `src/main/scala/tradebot/utils/BacktestWheelOptions.scala`

Wheel backtest specifics:
- Option universe based simulation.
- Optional synthetic contract fallback and model pricing modes.
- Assignment/called-away lifecycle simulation.
- Roll/profit-take logic aligned with live strategy intent.

Defined-risk backtest specifics:
- Simulates vertical and condor lifecycle.
- Uses trend/regime/VWAP/ADX/condor filters analogous to live strategy.
- Tracks trade PnL, drawdown, confidence summary.

Output and reporting:
- Symbol summaries, trades, confidence bands.
- Optional trade exports.

## 10. Business-Level Decision Framework

## 10.1 Wheel

Primary business levers:
- Premium capture quality vs fill quality.
- Assignment acceptance policy.
- Covered-call strike policy vs upside cap.
- Roll discipline (defensive vs income).
- Concentration control by symbol/sector.

Signal dimensions:
- Regime, LTF timing, EMA alignment, VWAP location, IV rank.

Risk dimensions:
- Gap risk in underlying.
- Volatility regime shifts.
- Slippage on market orders.
- Assignment and early exercise effects.

## 10.2 Defined-Risk

Primary business levers:
- Spread width and contracts (risk budget).
- Cooldown and filter strictness (trade frequency).
- Moneyness (credit vs probability profile).
- Condor enablement and range-bound strictness.

Signal dimensions:
- EMA trend shape, regime, VWAP deviation, ADX, time-of-day windows.

Risk dimensions:
- Credit spread adverse expansion.
- Correlated losses in trend reversals.
- Fill quality on combo market orders.

## 11. Operational Runbook

Pre-open:
- Confirm strategy toggles (`enable-wheel`, `enable-defined-risk`).
- Confirm symbol-to-strategy exclusivity in config.
- Confirm buying-power expectations for configured `block` sizes.
- Confirm event windows and daily trading boundaries.

Intraday:
- Watch strategy decision logs.
- Watch order-request logs and resolved option selection diagnostics (wheel).
- Watch pending-open/pending-roll/pending-close behavior.

Post-close:
- Reconcile fills, positions, option marks, and persisted order records.
- Compare realized behavior vs expected filters and cooldown.

## 12. Known Implementation Gaps and Trade-Offs

1. Wheel and defined-risk use market orders for final execution.
- Pros: simple and high fill probability.
- Cons: slippage and spread sensitivity.

2. No dedicated auto cancel/replace loop for non-terminal unfilled option orders.
- There is cancel capability in gateway protocol, but no strategy-level watchdog/reprice loop.

3. Backtest/live divergence remains possible.
- Backtest depends on available option history and/or model fallback.
- Live uses real-time quote quality and routing conditions.

## 13. Suggested Evolution Roadmap

Priority A:
- Introduce optional limit-based execution mode for wheel and defined-risk combos.
- Add timeout-based cancel/reprice policy for stuck `Submitted` orders.

Priority B:
- Add fill-quality analytics:
  - selection quote vs actual fill
  - spread-at-send vs slippage

Priority C:
- Expand scenario/stress harness:
  - volatility shock windows
  - low-liquidity windows
  - broker disconnect and reconnect robustness tests

## 14. Cross-Reference Documents

Existing focused docs:
- `documents/wheel-strategy-playbook.md`
- `documents/wheel-parameter-reference.md`
- `documents/defined-risk-trend-playbook.md`
- `documents/defined-risk-trend-reference.md`

This guide is the system-level glue document; the above files remain useful for strategy-only details and parameter quick reference.

