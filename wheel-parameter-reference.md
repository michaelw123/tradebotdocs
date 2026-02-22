# Wheel Parameter Reference

This document explains the parameters that affect Wheel strategy behavior in this codebase, what each parameter does, and where it is used.

## 0. Global Strategy Assignment Rule

In `tradebot.equities`, each symbol must be assigned to exactly one strategy block.

- Allowed blocks include `WheelStrategy`, `HybridStrategy`, and `DefinedRiskTrendStrategy`.
- If a symbol appears in multiple blocks, startup validation fails.

## 1. Source Of Truth And Override Order

1. `src/main/resources/application.conf` is the default source of parameters.
2. `AppConfig` loads and validates runtime config: `src/main/scala/tradebot/utils/AppConfig.scala`.
3. Backtest (`BacktestRunner`) uses config defaults, and command-line `key=value` overrides take precedence for that run.

Practical rule:
- No CLI override: backtest uses config.
- CLI override present: backtest uses CLI value.

## 2. Bar Construction (LTF/HTF)

These control the bar cadence used by indicators and wheel signals.

| Parameter | Current Value | Role | Used In |
|---|---:|---|---|
| `tradebot.trading-timeframe` | `5` min | Base LTF bar size target | `application.conf`, `AppConfig.indicators.tradingTimeframe` |
| `tradebot.indicators.htf-wheel-minutes` | `15` min | HTF size for wheel regime classification | `ConfigUtils.htfBarsForStrategy`, `EquityMarketDataActor` |

Implementation notes:
- Live feed is 5-second bars (`NewBar5s`) aggregated to LTF, then HTF: `src/main/scala/tradebot/actors/EquityMarketDataActor.scala`.
- HTF bar count for wheel is computed as `htf-wheel-minutes / trading-timeframe`.

## 3. Indicator Parameters Used By Wheel

### MACD
| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.macd.htf-fast-period` | `12` | HTF trend regime |
| `tradebot.macd.htf-slow-period` | `26` | HTF trend regime |
| `tradebot.macd.htf-signal-period` | `9` | HTF trend regime |
| `tradebot.macd.ltf-fast-period` | `8` | LTF entry timing |
| `tradebot.macd.ltf-slow-period` | `17` | LTF entry timing |
| `tradebot.macd.ltf-signal-period` | `6` | LTF entry timing |

Used by:
- `EquityIndicatorBuilder` and `MacdRolling`.
- HTF MACD participates in regime detection (`Bullish`, `Bearish`, `RangeBound`).
- LTF MACD participates in `LtfSignalDetector` pullback/reversion checks.

### RSI, ADX, Bollinger, MFI, ATR Percentile
| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.rsi.period` | `14` | LTF overbought/oversold + pullback/reversion logic |
| `tradebot.adx.period` | `14` | LTF range/trend strength in signal logic |
| `tradebot.bollingerbands.period` | `20` | LTF location vs bands |
| `tradebot.bollingerbands.k` | `2.0` | Band width |
| `tradebot.mfi.period` | `14` | LTF participation filter |
| `tradebot.atr-percentile.lookback` | `100` | Volatility regime filter |
| `tradebot.signal-filter.use-mfi-filter` | `true` | Enable MFI gate |
| `tradebot.signal-filter.mfi-long-min` | `45` | Long-side MFI threshold |
| `tradebot.signal-filter.mfi-short-max` | `55` | Short-side MFI threshold |
| `tradebot.signal-filter.use-atr-percentile-filter` | `true` | Enable ATR percentile gate |
| `tradebot.signal-filter.atr-percentile-min` | `15` | Minimum volatility percentile |
| `tradebot.signal-filter.atr-percentile-max` | `95` | Maximum volatility percentile |

Used by:
- `src/main/scala/tradebot/algos/LtfSignalDetector.scala`.

## 4. Regime Thresholds (Hardcoded)

These are not currently in `application.conf`.

| Parameter | Value | Role | File |
|---|---:|---|---|
| `TrendEnter` | `30` | ADX threshold to enter trend regime | `src/main/scala/tradebot/dto/EquityIndicatorBuilder.scala` |
| `TrendExit` | `20` | ADX threshold to exit trend regime | `src/main/scala/tradebot/dto/EquityIndicatorBuilder.scala` |

Meaning:
- `ADX >= 30` + HTF MACD alignment drives `Bullish/Bearish`.
- `ADX <= 20` pushes regime toward `RangeBound`.

## 5. Wheel Signal Gating (LTF Requirement)

| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.wheel.require-ltf-for-csp` | `true` | CSP opens require an LTF-allowed signal |
| `tradebot.wheel.require-ltf-for-cc` | `false` | CC opens require an LTF-allowed signal |

Used in:
- `wheel.decideWheelAction(...)`: `src/main/scala/tradebot/algos/wheel.scala`.

## 6. Wheel Roll / Profit-Take Parameters

| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.wheel.enable-roll` | `true` | Enables rolling behavior |
| `tradebot.wheel.roll-trigger-dte` | `5` | Defensive roll trigger window |
| `tradebot.wheel.min-dte` | `25` | DTE floor for new sells |
| `tradebot.wheel.max-dte` | `35` | DTE ceiling for new sells |
| `tradebot.wheel.target-dte` | `30` | Target DTE hint |
| `tradebot.wheel.income-roll-enabled` | `false` | Enable OTM income roll logic |
| `tradebot.wheel.income-roll-trigger-dte` | `12` | Income roll trigger window |
| `tradebot.wheel.profit-take-close-enabled` | `true` | Allows closing short option early |
| `tradebot.wheel.profit-take-price-fraction` | `0.20` | Close if option mark <= 20% of entry |
| `tradebot.wheel.profit-take-min-dte` | `3` | Earliest DTE for profit-take close |

Used in:
- Live: `src/main/scala/tradebot/actors/strategy/WheelStrategyActor.scala`.
- Backtest lifecycle: `src/main/scala/tradebot/utils/BacktestWheelLifecycle.scala`.

## 7. Pricing / Quality Guards For Option Selection

| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.wheel.target-open-premium-min-pct` | `0.55` | Minimum premium as % of underlying |
| `tradebot.wheel.target-open-premium-max-pct` | `0.90` | Maximum premium as % of underlying |
| `tradebot.wheel.min-open-option-price` | `0.30` | Absolute minimum premium |
| `tradebot.wheel.target-put-moneyness` | `0.95` | Put strike targeting |
| `tradebot.wheel.target-call-moneyness` | `1.05` | Call strike targeting |
| `tradebot.wheel.market-order-guard-enabled` | `true` | Spread guard for marketable entries |
| `tradebot.wheel.market-order-max-spread-abs` | `0.25` | Absolute spread cap |
| `tradebot.wheel.market-order-max-spread-pct` | `0.20` | Relative spread cap |
| `tradebot.wheel.quote-scan-batch-size` | `20` | Quote request batch size |
| `tradebot.wheel.quote-scan-inter-batch-ms` | `250` | Pause between quote batches |
| `tradebot.wheel.quote-scan-max-strikes` | `24` | Max strike candidates to scan |
| `tradebot.wheel.selection-put-target-abs-delta` | `0.22` | CSP strike delta target for scoring |
| `tradebot.wheel.selection-put-min-abs-delta` | `0.12` | CSP minimum allowed abs delta |
| `tradebot.wheel.selection-put-max-abs-delta` | `0.35` | CSP maximum allowed abs delta |
| `tradebot.wheel.selection-put-min-annualized-credit-pct` | `0.04` | CSP minimum annualized credit floor |
| `tradebot.wheel.selection-call-target-abs-delta` | `0.25` | CC strike delta target for scoring |
| `tradebot.wheel.selection-call-min-abs-delta` | `0.12` | CC minimum allowed abs delta |
| `tradebot.wheel.selection-call-max-abs-delta` | `0.40` | CC maximum allowed abs delta |
| `tradebot.wheel.selection-call-min-annualized-credit-pct` | `0.03` | CC minimum annualized credit floor |

Used by:
- Live order gating and quote scan logic.
- Backtest option contract selection and synthetic fallback pricing.

## 8. Trend / VWAP / IV-Rank Filters

| Parameter | Current Value | Role |
|---|---:|---|
| `tradebot.wheel.ema-trend-filter-enabled` | `true` | Enable EMA trend regime gating |
| `tradebot.wheel.ema-trend-strict-for-csp` | `true` | Stricter CSP entry in downtrend |
| `tradebot.wheel.ema-trend-strict-for-cc` | `true` | Stricter CC entry in uptrend |
| `tradebot.wheel.vwap-deviation-filter-enabled` | `true` | Enable VWAP deviation gating |
| `tradebot.wheel.vwap-deviation-min-pct-for-csp` | `-1.5` | Lower bound for CSP entry |
| `tradebot.wheel.vwap-deviation-max-pct-for-csp` | `0.5` | Upper bound for CSP entry |
| `tradebot.wheel.vwap-deviation-min-pct-for-cc` | `-0.5` | Lower bound for CC entry |
| `tradebot.wheel.vwap-deviation-max-pct-for-cc` | `1.5` | Upper bound for CC entry |
| `tradebot.wheel.iv-rank-filter-enabled` | `false` | Enable IV-rank gating |
| `tradebot.wheel.iv-rank-min-pct-for-csp` | `20.0` | Min IV-rank for CSP |
| `tradebot.wheel.iv-rank-max-pct-for-csp` | `90.0` | Max IV-rank for CSP |
| `tradebot.wheel.iv-rank-min-pct-for-cc` | `15.0` | Min IV-rank for CC |
| `tradebot.wheel.iv-rank-max-pct-for-cc` | `85.0` | Max IV-rank for CC |
| `tradebot.wheel.iv-rank-lookback-days` | `252` | Rolling lookback for IV-rank |

## 9. Symbol Overrides In Config

`tradebot.wheel-symbol-overrides` currently sets:
- `SPY`: `min-dte=21`, `max-dte=30`, `target-dte=25`, `target-put-moneyness=0.96`
- `QQQ`: `min-dte=21`, `max-dte=30`, `target-dte=25`, `target-put-moneyness=0.96`
- `TLT`: `min-dte=25`, `max-dte=35`, `target-dte=30`, `target-put-moneyness=0.95`

## 10. Backtest-Specific Knobs (Can Override Config)

Backtest adds simulation knobs (CLI optional):
- `optionPriceMaxGapMinutes`
- `wheelOptionPricingModel`
- `wheelModelVol`, `wheelModelRate`, `wheelModelDividendYield`, `wheelModelSteps`
- `wheelAllowSyntheticContractWhenNoChain`
- `wheelSyntheticTargetDte`
- `wheelHoldOvernight`
- `wheelSyntheticExpiryAtEnd`
- `forceCloseAtEnd`
- `slippageBps`
- `feePerOrder`

Main parser:
- `src/main/scala/tradebot/utils/BacktestRunner.scala` (`parseArgs`).

## 11. Operational Guidance

1. Keep live and backtest aligned by changing config first.
2. Use CLI overrides only for controlled experiments.
3. When comparing backtests, log full argument strings so differences are auditable.
4. If you change `trading-timeframe`, verify LTF aggregation behavior in live path before assuming full propagation.
