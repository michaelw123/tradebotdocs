# Wheel + Defined-Risk Architecture One-Pager

This is a compact visual companion to:
- `documents/wheel-defined-risk-implementation-guide.md`

## 1. Component View (Live)

```mermaid
flowchart LR
  TM[TradeBotMain] --> TBA[TradeBotActor]

  TBA --> EQMD[EquityMarketDataActor]
  TBA --> ST[StateActor]
  TBA --> ORD[OrderActor]
  TBA --> IB[IBKRGatewayActor]
  TBA --> PERS[PersistenceActor]

  EQMD --> WS[WheelStrategyActor]
  EQMD --> DRS[DefinedRiskTrendStrategyActor]
  EQMD --> EX[ExitStrategyActor]
  ST --> WS
  ST --> DRS
  ST --> EX

  WS --> ORD
  DRS --> ORD
  EX --> ORD

  ORD --> WOS[WheelOrderService]
  ORD --> DROS[DefinedRiskOrderService]
  ORD --> IB
  ORD --> PERS

  IB --> WSP[WheelSecDefPlanner]
  IB --> WQ[WheelQuoteScanMachine]
  IB --> WSEL[WheelOptionSelection]
  IB --> WOB[WheelOptionBook]
  IB --> DRCB[DefinedRiskConIdBook]

  IB --> TWS[(IBKR TWS/Gateway)]
  TWS --> IB
  IB --> EQMD
  IB --> ST
  IB --> ORD
```

Code map:
- `TradeBotMain`: `src/main/scala/tradebot/TradeBotMain.scala`
- Root wiring/lifecycle: `src/main/scala/tradebot/actors/TradeBotActor.scala`
- Equity market data + indicator fan-out: `src/main/scala/tradebot/actors/EquityMarketDataActor.scala`
- Shared state/positions/option marks/IV rank: `src/main/scala/tradebot/actors/StateActor.scala`
- Order orchestration: `src/main/scala/tradebot/actors/OrderActor.scala`
- Wheel order service: `src/main/scala/tradebot/actors/order/wheel/WheelOrderService.scala`
- Defined-risk order service: `src/main/scala/tradebot/actors/order/definedrisk/DefinedRiskOrderService.scala`
- IB gateway runtime: `src/main/scala/tradebot/actors/IBKRGatewayActor.scala`
- IB gateway protocol: `src/main/scala/tradebot/actors/IBKRGatewayProtocol.scala`
- IB gateway flow mapping: `src/main/scala/tradebot/actors/IBKRGatewayFlow.scala`
- Wheel sec-def planner: `src/main/scala/tradebot/actors/wheel/WheelSecDefPlanner.scala`
- Wheel quote scan machine: `src/main/scala/tradebot/actors/wheel/WheelQuoteScanMachine.scala`
- Wheel strike selection/model: `src/main/scala/tradebot/actors/wheel/WheelOptionSelection.scala`
- Wheel option quote book: `src/main/scala/tradebot/actors/wheel/WheelOptionBook.scala`
- Defined-risk conId pending book: `src/main/scala/tradebot/actors/definedrisk/DefinedRiskConIdBook.scala`
- Persistence actor: `src/main/scala/tradebot/actors/PersistenceActor.scala`
- DB adapter: `src/main/scala/tradebot/persistence/SlickPersistenceAdapter.scala`

## 2. Live Sequence (Wheel Open/Roll)

```mermaid
sequenceDiagram
  participant M as EquityMarketDataActor
  participant W as WheelStrategyActor
  participant O as OrderActor
  participant G as IBKRGatewayActor
  participant Q as WheelQuoteScanMachine
  participant S as WheelOptionSelection
  participant T as IBKR TWS
  participant ST as StateActor

  M->>W: NewBar(Entity with indicators/regime)
  W->>W: decideWheelAction + gating
  W->>O: WheelOpenOption / WheelRollOption
  O->>G: RequestWheelOption / RequestWheelRoll
  G->>T: reqContractDetails + reqSecDefOptParams
  T-->>G: underlying + option chain metadata
  G->>Q: plan + dispatch quote scan
  G->>T: reqMktData(batch strikes)
  T-->>G: bid/ask/last/mark ticks
  G->>Q: applyTick(...)
  G->>S: finalizeSelection(...)
  S-->>G: selected strike/price context
  G-->>O: PlaceResolvedWheelOption / PlaceResolvedWheelRoll
  O->>O: buy-power + market guard checks
  O->>T: SendOrder (via Gateway)
  T-->>O: order status/error callbacks
  T-->>ST: positions + option marks
  ST-->>W: StateUpdate (pending clears when confirmed)
```

Code map:
- Wheel strategy logic and pending gates: `src/main/scala/tradebot/actors/strategy/WheelStrategyActor.scala`
- Wheel decision primitives: `src/main/scala/tradebot/algos/wheel.scala`
- LTF signal generation: `src/main/scala/tradebot/algos/LtfSignalDetector.scala`
- Wheel order request + resolved order handling: `src/main/scala/tradebot/actors/OrderActor.scala`
- Wheel order final checks/submit: `src/main/scala/tradebot/actors/order/wheel/WheelOrderService.scala`
- Gateway option scan orchestration: `src/main/scala/tradebot/actors/IBKRGatewayActor.scala`
- Quote scan scoring and selection: `src/main/scala/tradebot/actors/wheel/WheelQuoteScanMachine.scala`
- Option model helpers and strike scoring: `src/main/scala/tradebot/actors/wheel/WheelOptionSelection.scala`
- Option quote cache + mark updates: `src/main/scala/tradebot/actors/wheel/WheelOptionBook.scala`

## 3. Live Sequence (Defined-Risk Entry/Exit)

```mermaid
sequenceDiagram
  participant M as EquityMarketDataActor
  participant D as DefinedRiskTrendStrategyActor
  participant O as OrderActor
  participant G as IBKRGatewayActor
  participant C as DefinedRiskConIdBook
  participant T as IBKR TWS
  participant ST as StateActor

  M->>D: NewBar(Entity with HTF/LTF indicators)
  D->>D: trend/condor filters + cooldown + pendingClose logic
  D->>O: DefinedRiskOpen/Close command
  O->>O: build contracts + persist order metadata
  O->>G: RequestDefinedRiskCombo / IronCondor
  G->>C: pending conId resolution state
  G->>T: reqContractDetails for each leg
  T-->>G: conId resolved
  G-->>O: PlaceResolvedDefinedRiskCombo / Condor
  O->>T: SendOrder (combo MKT via Gateway)
  T-->>O: order status/error callbacks
  T-->>ST: option positions/marks
  ST-->>D: StateUpdate (active spread/condor + pendingClose updates)
```

Code map:
- Defined-risk strategy entry/exit + condor filters: `src/main/scala/tradebot/actors/strategy/DefinedRiskTrendStrategyActor.scala`
- Defined-risk order creation and combo submit: `src/main/scala/tradebot/actors/order/definedrisk/DefinedRiskOrderService.scala`
- Defined-risk command routing and resolved placement: `src/main/scala/tradebot/actors/OrderActor.scala`
- Defined-risk conId resolution workflow: `src/main/scala/tradebot/actors/definedrisk/DefinedRiskConIdBook.scala`
- Gateway conId resolution and dispatch: `src/main/scala/tradebot/actors/IBKRGatewayActor.scala`

## 4. Backtest Dataflow

```mermaid
flowchart LR
  CLI[BacktestRunner CLI args] --> BR[BacktestRunner]
  CFG[AppConfig + symbol overrides] --> BR
  CSV[(Bar CSV)] --> BR
  OPT[(Option history)] --> BR
  IV[(IV history)] --> BR

  BR --> IND[EquityIndicatorBuilder]
  BR --> WBL[BacktestWheelLifecycle]
  BR --> WOPT[BacktestWheelOptions]
  BR --> DRSIM[Defined-Risk simulation path]
  IND --> WBL
  WOPT --> WBL

  WBL --> RES[CompletedTrade + SymbolSummary]
  DRSIM --> RES
  RES --> OUT[(results/report exports)]
```

Code map:
- Backtest main and CLI overrides: `src/main/scala/tradebot/utils/BacktestRunner.scala`
- Wheel backtest lifecycle: `src/main/scala/tradebot/utils/BacktestWheelLifecycle.scala`
- Wheel option history/model pricing for backtest: `src/main/scala/tradebot/utils/BacktestWheelOptions.scala`
- Backtest IO and reports:
  - `src/main/scala/tradebot/utils/BacktestDataIO.scala`
  - `src/main/scala/tradebot/utils/BacktestReporting.scala`
- Config source and symbol overrides:
  - `src/main/resources/application.conf`
  - `src/main/scala/tradebot/utils/AppConfig.scala`
  - `src/main/scala/tradebot/utils/ConfigUtils.scala`

## 5. Key Design Notes

- Wheel and defined-risk both execute via `OrderActor` + `IBKRGatewayActor`.
- Wheel has dedicated strike-selection/quote-scan modules before order submit.
- Defined-risk uses contract construction + conId resolution for vertical/condor combos.
- State closure loop is position/mark driven through `StateActor` updates.
- Backtest mirrors strategy intent but can diverge from live due to option history and fill assumptions.
