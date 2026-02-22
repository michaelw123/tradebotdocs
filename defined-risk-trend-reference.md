# Defined-Risk Trend Strategy Reference

This document explains how to configure and run `DefinedRiskTrendStrategy` in this codebase.

## 1. Symbol Assignment Rule

Each equity symbol must belong to exactly one strategy block in `tradebot.equities`.

- Valid: symbol appears in only one of `WheelStrategy`, `HybridStrategy`, `DefinedRiskTrendStrategy`.
- Invalid: symbol appears in multiple strategy blocks.

Startup validation will fail if duplicates are found (case-insensitive).

## 2. Config Layout

File: `src/main/resources/application.conf`

### Strategy switches

```hocon
tradebot.strategies {
  enable-defined-risk = true
}
```

### Strategy parameters

```hocon
tradebot.defined-risk {
  min-dte = 14
  max-dte = 35
  target-dte = 21
  spread-width = 5.0
  put-short-moneyness = 0.97
  call-short-moneyness = 1.03
  contracts-per-trade = 1
  cooldown-bars = 6
  ema-trend-strict = true
  vwap-deviation-filter-enabled = true
  vwap-deviation-min-pct-for-bull-put = -1.5
  vwap-deviation-max-pct-for-bull-put = 0.8
  vwap-deviation-min-pct-for-bear-call = -0.8
  vwap-deviation-max-pct-for-bear-call = 1.5
}
```

### Symbol list

```hocon
tradebot.equities: [
  {
    DefinedRiskTrendStrategy: [
      {
        symbol = "IWM"
        sec-type = "STK"
        currency = "USD"
        exchange = "SMART"
        block = 100
        adaptive-priority = "Normal"
      }
    ]
  }
]
```

## 3. Current Entry Behavior

- Bull trend setup: opens a bullish put vertical intent.
- Bear trend setup: opens a bearish call vertical intent.
- Uses regime + EMA trend + VWAP deviation filter.
- Applies per-symbol bar cooldown.
- Blocks new entries while an existing short option side is already open for that symbol.

## 4. Execution Behavior

- Current implementation submits spread legs as separate option market orders (short leg + hedge long leg).
- Buying-power guard uses estimated max loss: `spreadWidth * contracts * 100`.

## 5. Run

```bash
sbt run
```

Recommended paper test isolation:

1. `enable-defined-risk = true`
2. Disable other equity strategies for the same symbols (or fully disable while testing).
3. Start with one symbol, e.g. `IWM`.

