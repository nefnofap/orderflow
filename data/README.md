# Data folder

Drop historical OHLCV exports here for Python backtesting of the
Fabervaale AMT/ORB strategy.

## Expected CSV format

A header row plus one row per bar. Column order doesn't matter; these names are
recognised (case-insensitive):

```
datetime, open, high, low, close, volume
2026-02-17 09:30:00, 21850.25, 21861.00, 21848.50, 21858.75, 1234
2026-02-17 09:31:00, 21858.75, 21864.50, 21855.00, 21860.25, 980
...
```

Notes:
- `datetime` may be ISO text (UTC or ET) **or** a Unix `timestamp` column.
- Tell me which timezone the timestamps are in.
- 1-minute data is ideal (lets the backtester build a proper opening range and a
  better CVD proxy). 5-minute also works.
- More history = more trades = more meaningful statistics.

## Suggested filename

`nq_1m.csv`, `mnq_5m.csv`, `btcusdtperp_1m.csv`, etc.
