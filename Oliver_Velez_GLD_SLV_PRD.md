# Oliver Velez Intraday Trading System
## TradingView Indicator & Strategy for GLD and SLV

**Product Requirements Document v1.0**

| | |
|---|---|
| Project | OV-GLD-SLV-v1 |
| Target Platform | TradingView (Pine Script v5) |
| Instruments | GLD (SPDR Gold Trust), SLV (iShares Silver Trust) |
| Primary Timeframes | 2-min, 5-min, 15-min (intraday) |
| Session Focus | US Regular Trading Hours (09:30–16:00 ET) |
| Opening Window | First 20 minutes (09:30–09:50 ET) = highest priority |
| Source Material | 8 video transcripts from Oliver Velez |
| Purpose | Specification for Codex to generate Pine Script v5 |
| Status | Draft for implementation |

---

## 0. Critical Disclaimers (Read Before Implementation)

**This section is mandatory. Any implementer — human or AI — must preserve these caveats in code comments at the top of both generated `.pine` files.**

### 0.1 Source material is a single trader
All trading rules in this PRD derive from one instructor (Oliver Velez). The system has not been independently validated by statistical analysis on GLD/SLV historical data. Win-rate figures cited in source material (82%, 87%, 88–92%) are self-reported by the instructor and are not verified. The indicator/strategy built from this spec is an implementation of a pedagogical framework, not a proven edge.

### 0.2 Strategy outputs are not financial advice
The resulting TradingView indicator generates signals based on pattern-matching rules. Signals do not constitute recommendations to buy or sell. Past performance of these patterns does not guarantee future results. Users must paper trade before deploying real money.

### 0.3 Instrument-specific behavior differs from source examples
Source material primarily uses high-volatility single stocks (Tesla, Meta, Coinbase, Palantir, Pepsi). GLD and SLV have materially different volatility profiles, intraday range, gap behavior, and macro-driven dynamics. Thresholds labeled "fat bar" and "power bar" in the source are calibrated to single stocks and must be adapted to GLD/SLV using Average True Range (ATR) normalization, specified in Section 5.

### 0.4 Backtest before deploying
The strategy component MUST be backtested on at least 6 months of intraday data for both GLD and SLV before any real-capital deployment. Expected win rates, drawdowns, and R-multiples must be empirically measured, not assumed from source claims.

---

## 1. Product Overview

### 1.1 Goal
Build a TradingView indicator (and optional strategy) in Pine Script v5 that identifies Oliver Velez's intraday trading setups on GLD and SLV, displays them visually on the chart, and generates alerts for trade execution.

### 1.2 Two Deliverables

| Component | Type | Purpose |
|---|---|---|
| Indicator | Pine Script `indicator()` | Visual overlays, zone drawing, pattern detection, alerts. Read-only — does not place trades. |
| Strategy | Pine Script `strategy()` | Backtestable version with entry/exit/stop logic. For historical validation only. |

### 1.3 Non-Goals (Out of Scope for v1)
- Automated trade execution (no broker integration)
- Machine learning / adaptive thresholds
- Multi-symbol scanning (v1 runs per-chart)
- Options strategies, futures (GC/SI) support
- Pre-market / after-hours signal generation (regular session only)
- Mobile-specific optimizations

### 1.4 Success Criteria
- Indicator correctly draws the Fabulous Four zone before each session open
- Indicator correctly identifies and labels at least 80% of manually-verified pattern instances on a sample 20-day GLD chart and 20-day SLV chart
- Strategy backtest produces performance metrics over 6+ months of 2-min data without runtime errors
- All 10 signal types (Section 6) fire under specified conditions
- Code passes TradingView's built-in compiler without warnings

---

## 2. Glossary (Oliver Velez Terminology)

| Term | Definition |
|---|---|
| **Fab Four (Fabulous Four)** | Rectangular zone drawn pre-market using 4 reference prices: 200 SMA, 20 SMA, prior day close, and the high/low of the last 45 min of prior session. Box = [lowest of all 4, highest of all 4]. |
| **State (Narrow/Wide)** | Narrow = 20 SMA and 200 SMA are close together. Wide = they are far apart. Measured by normalized distance (see §5.2). |
| **Position 1 (below state)** | Price opens below the Fab Four / narrow state cluster. Bearish bias. |
| **Position 2 (inside state / "trap zone")** | Price opens inside the Fab Four zone. No-trade zone. |
| **Position 3 (above state)** | Price opens above the Fab Four / narrow state cluster. Bullish bias. |
| **Extreme Position** | Price opens very far above/below the state ("outside the firmament"). Fade candidate. See §6.1. |
| **Elephant Bar** | Solid bar (minimal wicks) with body size significantly larger than recent average. Quantified in §5.3. |
| **Tail Bar (bottoming/topping)** | Bar with a long lower wick (bottoming, bullish) or long upper wick (topping, bearish). Body small, wick large. §5.4. |
| **Bull 180 / Bear 180** | Two-bar reversal pattern. Bull 180: fat red bar immediately followed by a green bar whose high exceeds the red bar's high. Mirror for Bear 180. §6.3. |
| **Power Bar Start** | Large destructive bar that reverses direction of a prior color run. Body size exceeds threshold AND wipes out specified portion of prior opposite-color progress. §6.4. |
| **Three-Finger Spread** | Condition where price, 20 SMA, and 200 SMA are all widely separated in a specific order. §6.5. |
| **20 MA Halt** | Price was below 20 SMA for N bars, rallies through 20 SMA, pulls back to 20 SMA, and the pullback fails to close back below. §6.6. |
| **200 MA Surge** | Price originates near a flat 200 SMA and moves sharply away. §6.7. |
| **Trifecta** | A 200 MA Surge occurring simultaneously on 2-min, 5-min, AND 15-min timeframes in the same direction. §6.8. |
| **Market Law 4** | Failed retest pattern. After a trend of lower-lows (or higher-highs), price rallies (or drops), returns, and fails to make a significant new extreme. §6.9. |
| **Color Game** | Add-on entry rule. In an uptrend: after a single isolated red bar, enter when a subsequent green bar takes out the red's high. Mirror for downtrend. §7.2. |
| **3-Push Exit** | 3 distinct directional pushes (impulse waves) from entry = take profit signal. §8.1. |
| **Big-Bar Stop** | Trade management rule: when a large favorable bar forms, trail stop to just beyond that bar's opposite extreme. §8.3. |
| **Hidden Green/Red** | Bar that opens and initially trades in the "wrong" color for its location, then reverses within the same bar to eliminate that excursion. §6.10. |

---

## 3. System Architecture

### 3.1 Execution Model
The indicator runs per-bar on TradingView's standard execution model. On each new bar close, all detection logic re-evaluates. Intrabar detection (confirming signals before bar close) is supported via `barstate.isconfirmed` vs. `barstate.isrealtime` checks — see §10 for specifics.

### 3.2 Layer Architecture
The system is layered. Each layer depends only on layers below it. This ordering must be preserved in the Pine Script code organization.

| Layer | Name | Responsibility | Depends On |
|---|---|---|---|
| L0 | Context | Session detection, time-of-day filters, data flags | — |
| L1 | Indicators | 200 SMA, 20 SMA, 8 SMA, ATR, prior day OHLC, prior 45-min range | L0 |
| L2 | Fab Four | Compute and draw the Fab Four zone pre-open | L1 |
| L3 | State | Classify current state (narrow/wide), position (1/2/3) | L1, L2 |
| L4 | Bar Classifier | Label each bar: elephant, tail, fat, normal, power | L1 |
| L5 | Pattern Detector | Detect 180s, halts, surges, spreads, market law 4, color game | L1, L3, L4 |
| L6 | Signal Aggregator | Combine detections, apply location filter, emit signals | L3, L5 |
| L7 | Trade Manager (strategy only) | Entries, stops, adds, partials, trailing, exits | L6 |
| L8 | Visualization | Plots, shapes, labels, background colors | All |
| L9 | Alerts | `alertcondition()` calls for each signal type | L6 |

### 3.3 Input Parameters (TradingView UI)

All thresholds in this PRD are expressed as ATR multiples or lookback-relative ratios, NOT absolute dollar amounts. This is critical because GLD and SLV have different volatility and absolute prices.

| Parameter | Type | Default | Range | Description |
|---|---|---|---|---|
| `atr_length` | int | 14 | 5–50 | ATR lookback for volatility normalization |
| `fab4_late_day_minutes` | int | 45 | 15–90 | Minutes of prior session used for Fab Four zone |
| `narrow_state_atr_mult` | float | 0.75 | 0.25–2.0 | abs(20 SMA − 200 SMA) / ATR < this = narrow |
| `wide_state_atr_mult` | float | 2.5 | 1.5–5.0 | abs(20 SMA − 200 SMA) / ATR > this = wide |
| `elephant_body_atr_mult` | float | 1.5 | 1.0–3.0 | body / ATR > this = elephant |
| `elephant_wick_ratio_max` | float | 0.25 | 0.1–0.5 | total_wick / total_range ≤ this to qualify |
| `tail_wick_ratio_min` | float | 0.60 | 0.5–0.8 | tail_wick / total_range ≥ this to qualify |
| `fat_bar_body_atr_mult` | float | 1.2 | 0.8–2.5 | 180 pattern minimum body size threshold |
| `surge_atr_mult` | float | 2.0 | 1.0–4.0 | move from 200 SMA / ATR > this = surge |
| `spread_atr_mult` | float | 2.5 | 1.5–4.0 | Three-finger spread threshold |
| `push_count_exit` | int | 3 | 2–5 | Exit after N pushes (profit target) |
| `max_loss_per_trade_pct` | float | 1.0 | 0.25–3.0 | Max % of equity risked per trade (strategy only) |
| `opening_window_minutes` | int | 20 | 5–60 | Minutes after open that opening-bar rules apply |
| `enable_trifecta` | bool | true | — | Enable multi-timeframe confluence detection |
| `enable_color_game` | bool | true | — | Enable add-on signals via color game |

---

## 4. Pre-Market, Session, and Time Handling

### 4.1 Session Definition
GLD and SLV trade during US equity RTH (09:30–16:00 Eastern). The indicator operates only on RTH bars.

### 4.2 Session-Anchored Variables
Reset at each session open: session_high, session_low, bar_count_session, opening_bar_high, opening_bar_low, push_count (strategy), fab4_box.

### 4.3 Prior Session Data
Identify last bar of prior RTH session (16:00 close). Walk back N bars where N × timeframe_minutes = fab4_late_day_minutes. Record prior_late_high, prior_late_low, prior_close.

### 4.4 Opening Window Flag
in_opening_window true while bars_since_session_open × timeframe_minutes < opening_window_minutes. §6.1, §6.2, §6.10 fire only when true.

### 4.5 Gap Categorization (Position at Open)
At close of first session bar: Position 3 if open > fab4_top; Position 2 if fab4_bottom ≤ open ≤ fab4_top (no trade); Position 1 if open < fab4_bottom; Extreme if abs(open − fab4_mid) / ATR > 4.0.

---

## 5. Quantified Bar & State Definitions

### 5.1 Bar Anatomy
body = abs(close − open); total_range = high − low; upper_wick = high − max(open, close); lower_wick = min(open, close) − low; atr = ta.atr(atr_length).

### 5.2 State Classification
ma_separation = abs(sma20 − sma200); narrow_state = ma_separation < atr × narrow_state_atr_mult; wide_state = ma_separation > atr × wide_state_atr_mult.

### 5.3 Elephant Bar
ALL true: body > atr × elephant_body_atr_mult; (upper_wick + lower_wick) / total_range ≤ elephant_wick_ratio_max; body > 0. Green if close > open; red if close < open.

### 5.4 Tail Bar
Bottoming (bull): lower_wick / total_range ≥ tail_wick_ratio_min AND upper_wick / total_range ≤ 0.2 AND total_range > atr × 0.8. Topping (bear): mirror.

### 5.5 Flat 200 SMA
slope_200 = (sma200 − sma200[20]) / 20; is_flat_200 = abs(slope_200) < atr × 0.05.

### 5.6 Near/Away 20 SMA
near_20 = abs(close − sma20) < atr × 0.75; away_20 = abs(close − sma20) > atr × 2.0.

### 5.7 Near 200 SMA
near_200 = abs(close − sma200) < atr × 1.0.

### 5.8 Fat Bar
is_fat_bar = body > atr × fat_bar_body_atr_mult AND (upper_wick + lower_wick) / total_range < 0.4.

---

## 6. Setup Specifications (Signal Logic)

### 6.1 OV_POS3_FADE / OV_POS1_FADE (Opening Gap Fade)
Preconds: in_opening_window; bars_since_session_open ≤ 3; abs(session_open − fab4_mid) / atr in (3.0, 8.0). Trigger: bull 180, bear 180, elephant, or tail bar opposite to gap. Entry: 1 tick past trigger extreme in fade direction. Stop: 1 tick beyond other extreme. Targets: 1/3, 2/3, full of gap toward fab4_mid.

### 6.2 OV_NARROW_BREAK (Narrow State Opening Break)
Preconds: in_opening_window; narrow_state; opening bar pos 1 or 3 (not trap); not Extreme; opening bar is elephant or tail in correct direction. Trigger: opening bar close. Entry: 1 tick past opening bar extreme in breakout direction. Stop: 1 tick beyond opposite extreme.

### 6.3 OV_BULL_180 / OV_BEAR_180
Bull 180: bar[1] is_fat_bar AND red; bar[0] green AND high > high[1]; body[0] > body[1] × 0.75; consecutive (no intermediate red). Bear: mirror. Entry: high[1]+1tick (bull) / low[1]−1tick (bear). Stop: low[0]��1tick (bull) / high[0]+1tick (bear). 80% Entry variant: if body[1] > atr × 2.5, allow entry at low[1] + body[1] × 0.80 for bull. Location filter: near 20 → target away from 20; away from 20 → target back to 20; inside Fab Four → skip.

### 6.4 OV_POWER_BAR_BULL / OV_POWER_BAR_BEAR
Bear (reversing uptrend): for some N in 3..10, close − close[N] > atr × 1.0; current bar is red elephant; low < close[N] + (close − close[N]) × 0.5. Bull: mirror. Confirm at bar close. Entry: close of power bar OR next bar open. Stop: 1 tick beyond power bar extreme.

### 6.5 OV_THREE_FINGER_BULL / OV_THREE_FINGER_BEAR
Bear: close > sma20; sma20 > sma200; (close − sma20) > atr × spread_atr_mult; (sma20 − sma200) > atr × spread_atr_mult. Bull: mirror. Filter, not standalone trade. Stack: emit OV_THREE_FINGER_STACK when spread + (180 OR power bar) on same bar.

### 6.6 OV_HALT_LONG / OV_HALT_SHORT (20 MA Halt)
Bullish state machine S0..S4: S0 idle → close < sma20 for ≥10 consecutive bars → S1 extended_down. S1 → close crosses above sma20 and stays above ≥2 bars → S2 through. S2 → pullback low touches or dips below sma20 → S3 testing. S3 → close back above sma20 without closing > 25% ATR below → S4 halted; OR closes > 25% ATR below → S0 reset. S4 → green bar takes out high of most recent red bar → SIGNAL. Entry: 1 tick above recent red bar high. Stop: 1 tick below halt bar low. Bear: mirror.

### 6.7 OV_SURGE_UP / OV_SURGE_DOWN (200 MA Surge)
Bull: is_flat_200; for some N in 3..8, near_200[N]; close − sma200 > atr × surge_atr_mult; net upward movement (no major pullback). Trigger: first bar threshold met. Entry: signal close or next open. Stop: 1 tick beyond 200 SMA.

### 6.8 OV_TRIFECTA_UP / OV_TRIFECTA_DOWN (Multi-TF Surge)
request.security on '2','5','15' with lookahead=barmerge.lookahead_off. trifecta_bull = surge bull on all three TFs. trifecta_bear = mirror.

### 6.9 OV_MARKET_LAW_4_BULL / OV_MARKET_LAW_4_BEAR (Failed Retest)
Bull: prior trend ≥3 lower lows in last 20 bars; L1 low; rally ≥ atr × 2 from L1; pullback L2 with L2 ≥ L1 OR L2 < L1 by < atr × 0.3; rally from L2 with power bar or 180 confirming. Entry: per §6.3/§6.4. Stop: 1 tick below L2. Bear: mirror.

### 6.10 OV_HIDDEN_GREEN / OV_HIDDEN_RED (Opening Bar Reversal)
Hidden Red (shorts below Fab Four): in_opening_window AND bars_since_session_open == 1; opening bar below Fab Four; opens green and traded higher than open at some point; closes red (close < session_open). Trigger: opening bar close. Entry: 1 tick below opening bar low. Stop: 1 tick above green excursion high. Hidden Green: mirror.

### 6.11 Signal Stacking
S0 single; S1 any two on same bar; S2 trifecta TF alignment; S3 grand slam = three-finger spread + Market Law 4 + power bar/180 on same bar. Emit OV_STACK_S1, OV_STACK_S2, OV_STACK_S3.

---

## 7. Color Game (Strategy)

### 7.1 Concept
Add to winning positions when counter-color exhaustion is confirmed. Strategy only.

### 7.2 Long Add Rule
After long entry: track first red bar after entry as R1. If R1 followed by non-red bar with no further red, mark R1.high. When subsequent green bar high > R1.high, fire ADD_LONG. Add = 1/2 of remaining reserve. Stop unchanged. Skip if R1 occurs while away_20.

### 7.3 Short Add Rule
Mirror of §7.2.

### 7.4 Max Adds
Max 2 adds per trade.

### 7.5 Pattern Restart
After exit, color game restarts only when price returns to near_20.

---

## 8. Exit & Trade Management (Strategy)

### 8.1 Push Counting & 3-Push Exit
push = consecutive same-direction bars containing ≥1 bar with body > atr × 0.5. At entry push_count=0. Counter-color body-meaningful bar followed by another same-color push increments push_count. When push_count == push_count_exit, exit remainder.

### 8.2 Partial Exits
Push 1 → 25% off. Push 2 → 25% off + place limit for push-3 target on remaining 50%. Push 3 fill OR reverse → exit remainder.

### 8.3 Big-Bar Trailing Stop
After entry: any favorable bar with body > atr × 1.3 → move stop to that bar's opposite extreme ±1 tick (long: low−1tick; short: high+1tick), only if new stop is more favorable. Never lower a long stop / never raise a short stop.

### 8.4 Color-Pattern Trailing Stop
Long: pattern red-green-green at [2][1][0] → move stop to low of bar[2] − 1 tick (only if more favorable). Short: green-red-red → high of bar[2] + 1 tick.

### 8.5 State-Change Forced Exit
If entered narrow AND now wide AND away_20 → force exit at current bar close.

### 8.6 Hard Stop
Original entry stop persists unless superseded by tighter trailing stop. Never widen.

### 8.7 Time-Based Exit
For §6.1, §6.2, §6.10 entries: if no target/trail by opening_window_minutes + 10 minutes after open → flatten at market.

---

## 9. Visualization

### 9.1 Persistent Elements
Fab Four Box: semi-transparent blue fill, solid top/bottom lines, redrawn each session. Prior-day close: dotted gray horizontal line. Session VWAP: thin orange (optional).

### 9.2 Moving Averages
20 SMA blue 2px; 200 SMA red 2px; 8 SMA green dotted 1px.

### 9.3 Signal Markers
Bull 180: lime triangle up below. Bear 180: red triangle down above. Power bull: lime diamond below. Power bear: red diamond above. Halt: blue circle on. Surge: purple arrow. Trifecta: gold large star. Market Law 4: teal square at L2 opposite extreme. Three-finger spread: light yellow background tint. Stack S2+: larger marker + label. Trap zone open: red X on bar 1.

### 9.4 State Indicator Table (top-right)
Rows: current state (NARROW/TRANSITIONAL/WIDE); position (POS-1/POS-2/POS-3/EXTREME); in opening window (YES/NO); active signal count this session.

---

## 10. Pine Script v5 Implementation Notes

### 10.1 Repainting Prevention
All pattern confirmations on barstate.isconfirmed for alerts. All request.security() calls use lookahead=barmerge.lookahead_off. Strategy entries via strategy.entry(when=...) on confirmed conditions. Intrabar preview markers allowed on barstate.isrealtime but visually distinguished.

### 10.2 Historical Data Requirements
≥200 bars for 200 SMA; ≥20 bars for 20 SMA; Trifecta needs ≥200 HTF bars; Fab Four needs ≥1 prior session.

### 10.3 Performance
No unbounded loops in real-time paths. Use var for session-persistent state, reset on session.isfirstbar. Track and delete box/line objects per session. Use max_lines_count, max_boxes_count, max_labels_count in declaration.

### 10.4 Required Declarations
Indicator first lines:
//@version=5
indicator("Oliver Velez — GLD/SLV Intraday", shorttitle="OV-GLD-SLV", overlay=true, max_lines_count=500, max_boxes_count=500, max_labels_count=500)

Strategy first lines:
//@version=5
strategy("OV Strategy — GLD/SLV", shorttitle="OV-STRAT", overlay=true, initial_capital=50000, default_qty_type=strategy.percent_of_equity, default_qty_value=50, pyramiding=2, commission_type=strategy.commission.percent, commission_value=0.01)

### 10.5 Alert Conditions (exact names, must register all)
OV_BULL_180, OV_BEAR_180, OV_POWER_BAR_BULL, OV_POWER_BAR_BEAR, OV_HALT_LONG, OV_HALT_SHORT, OV_SURGE_UP, OV_SURGE_DOWN, OV_TRIFECTA_UP, OV_TRIFECTA_DOWN, OV_MARKET_LAW_4_BULL, OV_MARKET_LAW_4_BEAR, OV_THREE_FINGER_BULL, OV_THREE_FINGER_BEAR, OV_HIDDEN_GREEN, OV_HIDDEN_RED, OV_POS3_FADE, OV_POS1_FADE, OV_NARROW_BREAK_LONG, OV_NARROW_BREAK_SHORT, OV_STACK_S1, OV_STACK_S2, OV_STACK_S3.

### 10.6 Instrument Validation
Validate symbol is GLD or SLV. If not, display warning label; optionally disable signals via strict_instrument bool input (default false). Permitted: AMEX:GLD, ARCA:GLD, AMEX:SLV, ARCA:SLV.

---

## 11. Test Cases & Acceptance Criteria

### 11.1 Unit Tests
T1 Fab Four box matches max/min of 4 components and extends across session. T2 state classifier matches visual narrow/wide. T3 elephant detector ≥8/10 with ≤2 false positives. T4 tail detector same. T5 Bull 180: 5 visible cases all detected. T6 Bear 180 same. T7 20 MA Halt fires on post-rally pullback in 3 extended-decline scenarios. T8 Trifecta: all 3 TF flags co-fire. T9 Trap zone: no signals in opening window when open inside Fab Four. T10 Non-repaint: replay matches historical.

### 11.2 Integration Tests (Strategy)
I1 6 months GLD 2-min backtest completes. I2 same SLV. I3 max single-trade loss ≤ max_loss_per_trade_pct. I4 color game adds fire. I5 stops only trail favorably. I6 3-push exit closes position. I7 state-change forced exit. I8 opening-window time exit at +10 min.

### 11.3 Backtest Reporting
Total trades, win/loss rate, avg winner/loser R-multiple, max drawdown, profit factor, expectancy, breakdown by signal type.

---

*— End of PRD —*