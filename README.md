# Oliver Velez GLD/SLV Pine Scripts

## Files
- `oliver_velez_indicator.pine` — indicator build (PRD §2–§6, §9, §10)
- `oliver_velez_strategy.pine` — strategy build (indicator scope + PRD §7, §8)

## Load in TradingView
1. Open TradingView chart.
2. Open **Pine Editor**.
3. Paste one file at a time (`oliver_velez_indicator.pine` or `oliver_velez_strategy.pine`).
4. Click **Add to chart**.
5. For alerts, use **Create Alert** and choose any `OV_*` alertcondition name.

## Inputs and defaults (PRD §3.3)
- `atr_length` (14): ATR normalization length.
- `fab4_late_day_minutes` (45): Prior-session late-day window for Fab Four.
- `narrow_state_atr_mult` (0.75): Narrow state threshold.
- `wide_state_atr_mult` (2.5): Wide state threshold.
- `elephant_body_atr_mult` (1.5): Elephant body threshold.
- `elephant_wick_ratio_max` (0.25): Max wick ratio for elephant bars.
- `tail_wick_ratio_min` (0.60): Min wick ratio for tail bars.
- `fat_bar_body_atr_mult` (1.2): Fat bar threshold for 180 logic.
- `surge_atr_mult` (2.0): 200 MA surge threshold.
- `spread_atr_mult` (2.5): Three-finger spread threshold.
- `push_count_exit` (3): Strategy exit after N pushes.
- `max_loss_per_trade_pct` (1.0): Strategy risk budget per trade.
- `opening_window_minutes` (20): Opening window for opening setups.
- `gap_extreme_min_atr` (3.0): Minimum ATR-normalized opening gap for fade setups.
- `gap_extreme_max_atr` (8.0): Maximum ATR-normalized opening gap for fade setups.
- `enable_trifecta` (true): Enable 2m/5m/15m trifecta checks.
- `enable_color_game` (true): Enable strategy add-on entries.
- `opening_time_exit_extra_minutes` (10, strategy): Extra minutes added to opening-window bars when computing the forced time-exit limit for opening-window entries.

### GLD vs SLV recommendation
Use the same defaults first; then calibrate ATR-multiple thresholds (`elephant_body_atr_mult`, `fat_bar_body_atr_mult`, `surge_atr_mult`, `spread_atr_mult`) after instrument-specific backtests because SLV often exhibits different intraday volatility than GLD.

## Known Ambiguities
Every `// AMBIGUITY:` in code is listed here:
1. PRD allows AMEX/ARCA symbols; TradingView frequently reports ARCA as `NYSEARCA`, so validation treats `NYSEARCA` as ARCA-equivalent.
2. Prior late-day Fab Four range is defined in minutes; on coarse chart resolutions this implementation uses whichever bars overlap the target late-day window.
3. Hidden Green/Hidden Red requires intrabar path information not available on historical bars; implementation approximates excursion using OHLC (`high/low` relative to `open`).
4. Color Game reserve sizing in PRD is not fully specified; adds are implemented as half of ATR-risk size with max 2 adds.

## Acceptance test execution (PRD §11)
1. Compile each script in TradingView (expect no compile errors).
2. On GLD 2-min chart, verify Fab Four box at/after session open and moving averages plotted.
3. Scroll two recent sessions and confirm at least one `OV_*` marker/alert condition can be selected.
4. For strategy: run 30+ days GLD and SLV 2-min backtests; confirm orders and exits execute without runtime errors.
5. Load on non-permitted symbol (for example AAPL) and confirm warning label appears.
6. Use Bar Replay to compare historical/forward signal behavior for non-repaint sanity (confirmed-bar signaling).

## Alert locations
In TradingView **Create Alert**, script conditions include exact names from PRD §10.5:
`OV_BULL_180`, `OV_BEAR_180`, `OV_POWER_BAR_BULL`, `OV_POWER_BAR_BEAR`, `OV_HALT_LONG`, `OV_HALT_SHORT`, `OV_SURGE_UP`, `OV_SURGE_DOWN`, `OV_TRIFECTA_UP`, `OV_TRIFECTA_DOWN`, `OV_MARKET_LAW_4_BULL`, `OV_MARKET_LAW_4_BEAR`, `OV_THREE_FINGER_BULL`, `OV_THREE_FINGER_BEAR`, `OV_HIDDEN_GREEN`, `OV_HIDDEN_RED`, `OV_POS3_FADE`, `OV_POS1_FADE`, `OV_NARROW_BREAK_LONG`, `OV_NARROW_BREAK_SHORT`, `OV_STACK_S1`, `OV_STACK_S2`, `OV_STACK_S3`.
