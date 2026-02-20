# Wheel Strategy Playbook

## 1. Purpose and Scope

This document defines a complete, rules-based implementation of the wheel strategy for U.S. equity options.

The wheel is an options income strategy that cycles between:

1. Selling cash-secured puts (CSPs) on a stock you are willing to own.
2. Accepting assignment (if it happens) to acquire 100 shares per contract.
3. Selling covered calls (CCs) against assigned shares.
4. Repeating the cycle after shares are called away.

Primary objective: Generate risk-adjusted cash flow while controlling downside and concentration risk.

Secondary objective: Acquire shares at an effective discount (via put premium) and exit shares at a target sell price (via call strike plus premium).

## 2. Strategy Summary

## Core mechanics

- `Cash-Secured Put`: Sell a put while holding enough cash to buy 100 shares at strike.
- `Assignment`: If assigned, you buy 100 shares at strike (per contract).
- `Covered Call`: Sell a call against owned shares to collect additional premium.
- `Call-away`: If shares are called away, restart with CSPs.

## What the strategy is and is not

- It is a premium-harvesting and inventory-management strategy.
- It is not a hedge against large downside moves.
- It is not "free income"; losses can exceed total premium collected if stock declines materially.

## 3. Prerequisites

## Account and permissions

- Options approval for selling cash-secured puts and covered calls.
- Sufficient buying power and settlement awareness.
- Understanding of assignment risk and early exercise mechanics.

## Product constraints

- Standard U.S. equity option contract typically controls `100` shares.
- Most equity options are American style, so assignment can occur before expiration.

## Trader readiness

- Predefined risk limits.
- Written entry, management, and exit rules.
- Journal and review process.

## 4. Universe Selection (Underlying Screening)

Pick underlyings you are willing to hold through drawdowns.

## Hard filters

- Liquidity:
  - Average daily stock volume: typically `>= 1M` shares.
  - Option open interest near target strikes: typically `>= 500` contracts.
  - Tight bid/ask spread (stock and options).
- Option chain quality:
  - Multiple expirations with tradable spreads.
  - Sufficient strike granularity.
- Corporate/event risk:
  - Earnings dates known before entry.
  - Avoid unresolved binary events unless explicitly part of plan.

## Preference filters

- Stable, mature businesses over speculative names.
- Sector diversification to reduce correlated drawdowns.
- Avoid symbols with chronic gap risk unless compensated by position size.

## Exclusion examples

- Illiquid small-cap names with wide spreads.
- Meme-driven names with unstable implied volatility and jump risk.

## 5. Capital Allocation and Risk Limits

## Position sizing

- Per contract secured cash for CSP: `Strike x 100`.
- Max risk per contract is substantial if stock falls sharply.
- Keep unused cash reserve for roll flexibility and new opportunities.

## Portfolio guardrails (example policy)

- Max allocation per underlying: `5% to 10%` of net liquidation value (NLV).
- Max allocation per sector: `20% to 30%` of NLV.
- Max wheel exposure at portfolio level: set a cap (for example, `40% to 60%` of NLV).
- Minimum cash buffer: `20%+` depending on volatility regime.

## Risk of ruin controls

- Do not "average down" without a prewritten rule.
- Avoid stacking many contracts in correlated names.
- Limit new entries during macro volatility spikes unless position size is reduced.

## 6. Entry Rules for Cash-Secured Puts

## Timing

- Typical target expiration: `30 to 45 DTE` (days to expiration).
- Alternative shorter cycle: `14 to 28 DTE` for faster turnover (higher management intensity).

## Strike selection frameworks

Pick one framework and stay consistent:

- Delta-based: short put delta around `0.15 to 0.30`.
- Probability-based: strike with high probability of expiring OTM.
- Technical-based: strike below support zone you accept owning.

## Premium thresholds

- Minimum acceptable premium as percentage of secured cash (example rule):
  - `Premium / (Strike x 100) >= X%` for the selected DTE bucket.
- Reject trades where spread/slippage consumes too much edge.

## Event filter

- Decide upfront whether to hold short puts through earnings.
- If not holding through earnings:
  - Close or roll before event window.

## Entry checklist

- Underlying passes liquidity checks.
- Position size within limits.
- Earnings/dividend calendar reviewed.
- Exit/management actions pre-defined.

## 7. Management Rules for Short Puts

## Profit-taking

Common rule sets:

- Close at `50%` of max profit before expiration.
- Or hold to `21 DTE`, then close/roll to reduce gamma risk.

## Defensive adjustments

If stock declines toward/below strike:

- Roll out in time (same strike or lower strike) for net credit when possible.
- Avoid rolling purely to delay loss without improving expected outcome.
- If assignment is acceptable and near ex-dividend or expiry, allow assignment per plan.

## Assignment planning

- Assignment can occur before expiration for American-style equity options.
- Be especially alert near ex-dividend dates for ITM contracts.

## 8. Transition to Covered Calls (After Assignment)

## Cost basis tracking

Track:

- Share purchase cost from assignment.
- Total put premiums collected.
- Net adjusted stock basis.

Formula:

`Adjusted Basis = Assignment Price - (Total Net Put Premium per share)`

## Initial covered call setup

- Usual DTE: `30 to 45`.
- Strike selection:
  - At or above adjusted basis for non-loss realization preference.
  - Or lower strike if maximizing premium and accepting potential realized loss is intentional.

## Covered call management

- Profit target: buy back at `50% to 75%` of premium captured.
- If stock rallies through strike:
  - Let shares be called away if exit price meets plan.
  - Or roll up/out for net credit if retaining shares is priority.
- If stock falls:
  - Continue selling calls, but avoid strikes that lock in unacceptable loss unless policy allows.

## 9. Full Cycle Math and Performance Accounting

## Per-cycle return components

Total cycle P/L includes:

1. Put premium(s).
2. Stock P/L from assignment to call-away (or mark-to-market if still held).
3. Covered call premium(s).
4. Fees/slippage/borrow costs (if any).

## Key metrics

- Net premium income.
- Realized P/L by symbol.
- Annualized return on secured capital.
- Win rate by cycle (with caution; expectancy matters more).
- Average days in trade.
- Max drawdown (portfolio and per underlying).
- Assignment rate.

## Practical annualization

Use conservative annualization that includes idle cash and drawdowns, not just per-trade headline yield.

## 10. Risk Analysis

## Primary risks

- Downside equity risk: Stock can drop far below strike.
- Gap risk: Overnight moves bypass adjustment opportunities.
- Opportunity cost: Covered calls cap upside.
- Volatility regime shifts: Premium may not compensate tail risk.
- Liquidity/slippage risk in fast markets.

## Scenario examples

- Slow grind down:
  - Repeated call sales may not offset persistent equity decline.
- Sharp crash:
  - Multiple assignments can create concentrated unrealized losses.
- Sharp rally:
  - Calls cap upside versus buy-and-hold.

## Mitigation

- Strict diversification and per-name caps.
- Avoid weak/binary underlyings.
- Use event filters.
- Keep reserve cash and avoid maxing account buying power.

## 11. Tax, Compliance, and Operational Notes (U.S.-Focused)

- Option tax treatment can vary by holding period, assignment/exercise outcomes, and product type.
- Qualified covered call and straddle-related rules may affect holding-period and loss treatment in specific cases.
- Maintain complete records of premiums, assignments, and cost basis adjustments.
- Coordinate with a qualified tax professional before production deployment.

## Operational controls

- Pre-market check: earnings, ex-dividend, macro events.
- End-of-day reconciliation: fills, Greeks, exposure, buying power.
- Weekly review: performance attribution and rule adherence.

## 12. Standard Operating Procedure (SOP)

## Daily workflow

1. Scan watchlist and liquidity.
2. Review event calendar (earnings/dividends/news).
3. Open new CSPs only if limits permit.
4. Manage open puts/calls per rule triggers.
5. Log every action with rationale.

## Weekly workflow

1. Rebalance sector/position concentration.
2. Evaluate assignment candidates and call strike plans.
3. Audit slippage versus mid-price assumptions.
4. Review deviations from rules and corrective actions.

## Monthly workflow

1. Compute strategy KPIs.
2. Review drawdowns and stress performance.
3. Update parameter ranges only if statistically justified.

## 13. Parameter Pack (Example Defaults)

Use this as a baseline and tune with backtests/forward results.

- Universe: liquid large/mid-cap equities and broad ETFs.
- CSP DTE: `30 to 45`.
- CSP delta: `0.20` target.
- Take-profit:
  - Puts at `50%` max profit.
  - Calls at `50% to 75%` max profit.
- Management point:
  - Review/adjust at `21 DTE`.
- Max per underlying: `<= 7.5%` NLV.
- Max per sector: `<= 25%` NLV.
- Cash reserve: `>= 25%` NLV.
- Earnings policy: no new short premium positions inside defined blackout window.

## 14. Trade Log Template

Use one row per order event.

| Date | Symbol | Leg Type | Action | Expiry | Strike | Qty | Fill | Mid at Fill | Delta | IV Rank | DTE | Premium In/Out | Net Credit (Cycle) | Buying Power Impact | Reason Code | Notes |
|---|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|---|

Reason code examples:

- `ENTRY_CSP`
- `TAKE_PROFIT_PUT`
- `ROLL_DOWN_OUT_PUT`
- `ASSIGNMENT`
- `ENTRY_CC`
- `ROLL_CC`
- `CALLED_AWAY`

## 15. Decision Trees

## Short put at 21 DTE decision

1. Is profit target reached?
   - Yes: close.
   - No: continue.
2. Is underlying still thesis-valid and within risk budget?
   - No: close/reduce.
   - Yes: continue.
3. Is strike challenged?
   - No: hold/manage.
   - Yes: roll for credit or accept assignment per plan.

## Covered call decision

1. Is call near max profit?
   - Yes: close early or let expire per liquidity/time.
2. Is stock above strike and do you want to keep shares?
   - No: allow call-away.
   - Yes: roll up/out for acceptable credit/debit policy.
3. Is stock below adjusted basis?
   - Sell calls at strikes that do not violate your loss policy.

## 16. Failure Modes and Anti-Patterns

- Chasing high IV names without respecting tail risk.
- Ignoring assignment and ex-dividend dynamics.
- Oversizing in one ticker/sector.
- Rolling indefinitely without improving expected value.
- Evaluating only premium income while ignoring unrealized equity losses.

## 17. Backtesting and Validation Requirements

Before scaling:

1. Test across multiple market regimes (bull, bear, crash, sideways).
2. Include realistic slippage and commissions.
3. Model assignment possibilities, not just expiration outcomes.
4. Report drawdown depth/duration and capital utilization.
5. Run out-of-sample and forward paper validation.

## 18. Documented Assumptions

- U.S. listed equity options.
- American-style assignment risk applies.
- One contract generally represents `100` shares unless adjusted.
- Strategy operator accepts equity ownership risk.

## 19. Required Disclosures (If Shared Externally)

- "For educational purposes only; not investment, legal, or tax advice."
- "Options involve risk and are not suitable for all investors."
- Include balanced upside/downside language in all external communication.

## 20. Source Notes

Key mechanics and risk/disclosure references:

- OCC equity options product specifications: https://www.theocc.com/Clearance-and-Settlement/Clearing/Equity-Options-Product-Specifications
- FINRA options risks (assignment/dividend risk): https://www.finra.org/investors/investing/investment-products/options
- IRS Publication 550 overview and current publication page:  
  https://www.irs.gov/forms-pubs/about-publication-550  
  https://www.irs.gov/publications/p550

Update this section whenever broker, exchange, OCC, or IRS guidance changes.
