# Codex Build Instructions

## Task
Read the attached file `Oliver_Velez_GLD_SLV_PRD.md` and generate a complete TradingView Pine Script v5 implementation of the system described.

## Required Deliverables

Generate three files:

### 1. `oliver_velez_indicator.pine`
A TradingView indicator implementing **Sections 2–6, 9, and 10** of the PRD.

**Scope:**
- Pre-market Fab Four construction and drawing (§4.3, §9.1)
- All moving averages, ATR, and bar classification (§5)
- All 10 signal detection patterns (§6.1–§6.10) with alert emission
- Signal stacking logic (§6.11)
- Visualization: markers, zones, state indicator table (§9)
- All `alertcondition()` calls listed in §10.5

**Scope does NOT include:** entries, exits, position sizing (those belong to the strategy file).

### 2. `oliver_velez_strategy.pine`
A TradingView strategy for backtesting, implementing **everything in the indicator PLUS Sections 7 and 8**.

**Additional scope beyond the indicator:**
- Color Game add-on logic (§7)
- Push counting and 3-push exit (§8.1–§8.2)
- Big-bar trailing stop (§8.3)
- Color-pattern trailing stop (§8.4)
- State-change forced exit (§8.5)
- Hard stop preservation (§8.6)
- Time-based exit for opening-window signals (§8.7)
- Position sizing per `max_loss_per_trade_pct` input

### 3. `README.md`
A short markdown file containing:
- How to load each file into TradingView
- Description of each user input parameter and recommended defaults for GLD vs. SLV
- Known limitations (especially any ambiguities in the PRD you had to resolve)
- How to run the acceptance tests from §11
- Where to find alerts in TradingView's alert dialog

---

## Hard Requirements

1. **Pine Script v5 only.** First line of each `.pine` file must be `//@version=5`.

2. **Preserve all disclaimers from PRD Section 0** as code comments at the top of both `.pine` files. They are non-negotiable.

3. **Non-repainting.** Follow §10.1 strictly. Any `request.security()` call must use `lookahead=barmerge.lookahead_off`. All pattern confirmations must use `barstate.isconfirmed`.

4. **All thresholds must be user inputs** using `input.int()`, `input.float()`, `input.bool()`. Hardcoding is prohibited for anything in the §3.3 table.

5. **Never invent rules.** If the PRD is ambiguous or incomplete on any detail, DO NOT guess. Instead:
   - Add a code comment: `// AMBIGUITY: [description of what is unclear and what interpretation you chose]`
   - Pick the most conservative interpretation
   - List the ambiguity in `README.md` under a "Known Ambiguities" section

6. **Compile cleanly.** Before delivering, mentally walk through the code and ensure it compiles on TradingView without warnings.

7. **Instrument validation.** Implement §10.6 — check that the chart symbol is GLD or SLV and warn otherwise.

8. **Code organization.** Follow the layer architecture in §3.2. Group code into these sections with comment headers:
   ```
   // ─── L0: Context ───
   // ─── L1: Indicators ───
   // ─── L2: Fab Four ───
   // ─── L3: State ───
   // ─── L4: Bar Classifier ───
   // ─── L5: Pattern Detector ───
   // ─── L6: Signal Aggregator ───
   // ─── L7: Trade Manager ───     (strategy only)
   // ─── L8: Visualization ───
   // ─── L9: Alerts ───
   ```

---

## Implementation Order (suggested)

Build the indicator first in this order, testing each layer before moving to the next:

1. L0 + L1: Session/time flags + moving averages + ATR
2. L2: Fab Four box drawing
3. L3: State classification + Position at open
4. L4: Bar classifier (elephant, tail, fat, power)
5. L5: Start with Bull 180 and Bear 180 (§6.3) — simplest patterns
6. L5: Add Power Bar Start (§6.4), then 20 MA Halt state machine (§6.6)
7. L5: Add 200 MA Surge (§6.7), then Trifecta (§6.8) — most complex
8. L5: Add remaining patterns (§6.1, §6.2, §6.5, §6.9, §6.10)
9. L6: Signal aggregator + stacking (§6.11)
10. L8 + L9: Visualization and alerts
11. Then copy to strategy file and add L7 trade management

---

## What to Do If You Get Stuck

- If a specific Pine Script v5 API is unclear, prefer the most recent TradingView documentation patterns (as of 2024–2025).
- If a PRD rule seems to conflict with Pine Script limitations (e.g., intrabar detection vs. `barstate.isconfirmed`), document the conflict in `README.md` and choose the safer option (confirmed bar close).
- Do not extend the specification with features not in the PRD, even if they seem like improvements. This is v1.

---

## Acceptance Test

Before declaring the build complete, verify that:

- [ ] Both `.pine` files compile on TradingView
- [ ] Loading the indicator on a GLD 2-min chart renders the Fab Four box at the 09:30 ET session open
- [ ] At least one alert type fires within 2 days of historical data
- [ ] The strategy backtest completes without runtime errors on 30 days of GLD 2-min data
- [ ] Running the code against a chart of `AAPL` (non-permitted symbol) displays a warning

Deliver the three files with a brief summary of any ambiguities encountered and how you resolved them.
