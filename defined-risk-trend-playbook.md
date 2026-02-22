# Defined-Risk Trend Playbook

## 1. Scope

This playbook is for the `DefinedRiskTrendStrategy` implemented in this repo, with an initial focus on `IWM`.

Goals:
- Capture directional trend premium with limited-risk verticals.
- Keep behavior operationally simple and measurable in paper/live runs.

## 2. Strategy Definition

- Bullish setup: short put vertical (credit spread).
- Bearish setup: short call vertical (credit spread).
- Entry filters:
  - Regime + EMA trend alignment.
  - Optional VWAP deviation band checks.
  - Bar-based cooldown.
- One symbol belongs to exactly one strategy block in config.

## 3. Configuration Baseline (IWM)

Recommended baseline from current sweeps:
- `defined-risk.spread-width = 10`
- `defined-risk.cooldown-bars = 12`
- `defined-risk.put-short-moneyness = 0.96`
- `defined-risk.call-short-moneyness = 1.04`

Sizing:
- Contracts are derived from symbol `block` in `tradebot.equities`.
- Rule in code: `contracts = max(1, round(block / 100))`.
- Examples:
  - `block = 100` -> `1` contract
  - `block = 200` -> `2` contracts

## 4. Risk Sizing Rules

Per-spread max risk approximation:

`maxLoss ~= (spreadWidth - entryCredit) * 100 * contracts`

Example (`width=10`):
- 1 contract: roughly `$750-$900` max risk (credit-dependent).
- 2 contracts: roughly `$1,500-$1,800` max risk.

Portfolio guardrails:
- Max risk per spread: `<= 0.5% to 1.0%` of account.
- Start with `block=100` (1 contract) for 2-4 weeks.
- Increase to `block=200` only after stable execution and acceptable drawdown.

## 5. Entry/Exit Operating Rules

Entry:
- Open only when strategy filters pass.
- Do not force entries outside strategy signals.

Exit hierarchy:
1. Expiry reached.
2. Profit take (modeled by spread mark improvement).
3. Hard stop (spread mark deterioration).
4. Opposite trend signal / regime shift.
5. EOD force close (if configured).

## 6. Live/Paper Checklist

Before session:
1. Confirm `enable-defined-risk=true`.
2. Confirm target symbols are only under `DefinedRiskTrendStrategy`.
3. Confirm `block` is set to intended contract count.
4. Confirm buying power buffer is adequate.

During session:
1. Watch for entry logs from `DefinedRiskTrendStrategyActor`.
2. Verify orders are placed with expected strikes/expiry.
3. Verify no overlap with wheel/hybrid on same symbol.

After session:
1. Review fills and slippage vs expectation.
2. Review drawdown and open risk.
3. Record anomalies and parameter change requests.

## 7. Backtest Commands

Base IWM run:

```bash
sbt "runMain tradebot.utils.BacktestRunner symbol=IWM dir=backtest"
```

With parameter overrides:

```bash
sbt "runMain tradebot.utils.BacktestRunner symbol=IWM dir=backtest definedRiskSpreadWidth=10 definedRiskCooldownBars=12 definedRiskPutShortMoneyness=0.96 definedRiskCallShortMoneyness=1.04 feePerOrder=1.50"
```

## 8. Change Management

When changing parameters:
1. Change one dimension at a time where possible.
2. Re-run backtest with same date span for apples-to-apples comparison.
3. Promote to paper only after backtest + log sanity checks.
4. Promote to larger size only after paper stability window is complete.

## 9. Do/Do Not

Do:
- Keep symbol ownership per strategy exclusive.
- Keep risk sizing explicit and consistent.
- Track realized + unrealized behavior, not just win rate.

Do not:
- Increase size because of short winning streaks.
- Run multiple strategy actors on the same symbol.
- Ignore execution quality (leg risk/slippage) when judging results.

