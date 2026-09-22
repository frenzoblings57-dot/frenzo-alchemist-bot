
# Alchemist Signal Bot

Level 1 crypto/forex signal bot built on the **Alchemist strategy** (custom SMC/ICT framework). Signal-only — no auto-trading. Delivers formatted trade signals to Telegram once a full setup is confirmed.

## Status

Strategy fully specified. Execution/deployment (live data feed, Telegram delivery, backtesting) paused in favor of getting the full rulebook documented first. This README is the source of truth for the strategy logic.

---

## Markets

- BTC/USD
- XAU/USD (Gold)
- XAG/USD (Silver)
- EUR/USD, GBP/USD, USD/JPY, USD/CHF, USD/CAD, AUD/USD, NZD/USD

## Timeframe hierarchy

- 1D Key Level → 4H execution timeframe
- 4H Key Level → 1H execution timeframe

## Universal validation rule (applies to all 4 KL types)

A pattern forming (OCL origin, Classic A/V two-candle pattern, QM structural break, or S/R flip break) only creates a **candidate** Key Level. The candidate only becomes a **confirmed HTF KL** when price returns and **rejects** it:

- Wick touches/pierces the level
- Candle **body closes without breaking through** it

That rejection candle is the same event as the "HTF KL rejection" step below — confirmation and rejection are one and the same. If the body closes through instead, the candidate is invalidated.

---

## 1. Key Level Types

### 1.1 Open/Close Level (OCL)

- The level price is the **open or close of a candle body** (not the wick/high-low).
- **Multi-candle origin:** a tight consolidation where multiple candle bodies cluster at nearly the same open/close price.
- **Single-candle origin:** one candle's open/close, provided it's immediately followed by a sharp displacement/expansion candle.
- Validation requires the move away to cause a structural **BOS/CHoCH** (break a prior swing high/low), with strong displacement or a clean FVG. No fixed pip-count requirement.

### 1.2 Classic A (resistance) / Classic V (support)

- **Classic A:** Candle 1 bullish → Candle 2 bearish (up then down). KL price = Candle 1's close (the OCL boundary between the two candles).
- **Classic V:** Candle 1 bearish → Candle 2 bullish (down then up). KL price = Candle 1's close.
- Candle 2 must show **displacement** (body ≥ 50–60% of range, or exceeds recent ATR) and ideally break an LTF swing point or leave an FVG. A small/doji candle 2 doesn't qualify.
- **Gap handling:** if candle 1's close ≠ candle 2's open, candle 1's close takes priority as the KL price.

### 1.3 QM (Quasimodo)

**Bearish QM (short):**
1. Swing High (H)
2. Swing Low (L)
3. "Head" = a later high that **exceeds H**, sweeping buy-side liquidity — if it doesn't exceed H, this isn't QM
4. Structural break: price breaks below L (CHoCH/MSS)
5. QML price = **H's level** (the left-shoulder swing, before the Head)
6. Retest of QML from below → rejection → confirmed KL → drop to LTF for CHoCH
7. SL = beyond the Head's extreme (+ spread)

**Bullish QM (long):** full mirror — Low(L) → High(H) → Head is a lower low sweeping sell-side liquidity below L → break above H → QML = L's level → retest from above → rejection → confirmed KL.

### 1.4 S/R Flip (RBS / SBR)

- **RBS (resistance becomes support):** a prior Classic A's OCL gets broken with a displacement candle body-closing above it. Later, price retests that same OCL from above and it now acts as support.
- **SBR (support becomes resistance):** mirror, using a prior Classic V's OCL, broken down, retested from below as resistance.
- Break confirmation needs displacement (body ≥ 50–60% of range), ideally leaving an FVG at the flipped OCL.
- **Timeframe split:** 4H/Daily HTF locates the macro level and confirms the break; 1H LTF is used for the pullback retest/entry (1H CHoCH or rejection candle).
- A liquidity sweep is **not required** for a base valid RBS/SBR, but sweeping opposite-side liquidity right before the break upgrades it to a high-probability **"Breaker Block"** tier setup.

---

## 2. Liquidity

### Induced Liquidity (Inducement / IDM)
A minor swing point/pullback sitting just in front of an HTF Key Level. Retail mistakes it for real S/R; their stops get swept to fuel the final push into the true HTF KL. Acts as the final filter before touching an OCL/Classic A/V level.

### Engineered Liquidity (EQH / EQL / trendline)
A clean, deliberately-built pattern over time — Equal Highs, Equal Lows, or a trendline — that accumulates a large pool of retail stops/breakout orders (a Draw on Liquidity), which price later sweeps before reversing.

- **2 touches** = valid EQH/EQL, base confidence
- **3+ touches** = strong EQH/EQL, high confidence
- Tolerance: ~0.1–0.15% of price or a fraction of ATR (tune during paper-testing)

---

## 3. HTF Sweep + Rejection / LTF CHoCH

**HTF setup (long example):**
1. Bullish HTF KL at support/demand
2. Sell-side liquidity around/above the level (inducement/engineered)
3. Price drops into the KL, sweeps the liquidity, rejects with a wick
4. HTF candle body closes back above the KL (a body break/close through = invalid)
5. Mark the rejection candle → drop to LTF

**CHoCH (operational definition):**
- Bullish: LTF structure is initially bearish, a new LTF low forms, then price breaks the most recent valid LTF lower high — **confirmed by candle body**, not just a wick.
- Bearish: mirror.

Short setups mirror all of the above.

---

## 4. Entry / SL / TP

**Entry:**
- Bullish: limit order at the **top boundary (proximal line)** of the 1H order block/OCL/imbalance sitting directly below the 1H inducement low.
- Bearish: limit order at the **bottom boundary** of the 1H order block/OCL/imbalance sitting directly above the 1H inducement high.
- (Proximal line chosen over 50%/midpoint fill for maximum fill probability + tight invalidation.)

**Stop Loss:**
- `base_SL` = the extreme HTF (4H/Daily) rejection wick
- `buffer` = 0.15 × ATR(14) on the 1H timeframe
- Bullish: `SL = base_SL − buffer` / Bearish: `SL = base_SL + buffer`

**Take Profit (partial + trail):**
- **TP1** = nearest 1H/4H swing point or internal liquidity pool → on hit: close 50% position, move SL to breakeven
- **TP2** = major HTF draw on liquidity (EQH/EQL or Daily/4H structural extreme) → remaining 50% runs to this target

**R:R filter (signal quality gate):**
```

DISCARD signal if TP1_RR < 1:3 OR TP2_RR < 1:5
SEND signal only if TP1_RR >= 1:3 AND TP2_RR >= 1:5
```
Practical TP2 range: 1:5 up to ~1:10.

---

## 5. Signal Format (Telegram)

```
🔔 ALCHEMIST SIGNAL | GRADE A+

Pair: XAU/USD
Direction: 🟢 LONG
Setup Type: Classic V (Support)
Timeframe Alignment: 1D HTF → 1H LTF

Entry: 4,092.50
SL: 4,081.20
TP1: 4,108.00 (Close 50%, Move SL to BE)
TP2: 4,124.80 (Main Draw on Liquidity)

R:R (TP1): 1:1.37
R:R (TP2): 1:2.86

Confluence Checklist:
✅ Grade A+ Setup (Breaker Block + Sweep)
✅ HTF 1D Rejection Confirmed
✅ 1H IDM Swept before Entry
✅ Engineered Liquidity (3-Touch EQH) at TP2
✅ 1H CHoCH Confirmed

Risk Management: 1% - 2% Risk per trade
Time: <timestamp>
```

Open question: exact grading logic (how many confluence checks = A+ vs A vs B) not yet finalized.

---

## 6. Planned architecture (data + delivery)

```

Bybit API (BTC candles — no key needed)
Twelve Data free tier (XAU, XAG, forex pairs — 800 calls/day)
        ↓
Python Alchemist engine (rules above)
        ↓
Telegram Bot API (free, unlimited) → personal chat
```

- **TradingView**: not used in the pipeline — webhook alerts require a paid plan and Pine Script can't run the full custom multi-timeframe logic. TradingView charts are for manual visual reference only.
- **WhatsApp**: dropped in favor of Telegram — Meta ended free-tier WhatsApp messaging (Oct 1 2026), Telegram's Bot API is free and unlimited.
- Telegram bot (`@frenzo_alchemist_bot`) created and tested — sends formatted signal successfully.

## 7. Build order (from original plan)

1. Market-data connection — *not yet wired up live*
2. TradingView integration — *dropped, manual reference only*
3. KL detection — **fully specified** (this doc, section 1)
4. Induced/engineered liquidity detection — **fully specified** (section 2)
5. HTF sweep + rejection — **fully specified** (section 3)
6. CHoCH detection — **fully specified** (section 3)
7. Entry/SL/TP calculation — **fully specified** (section 4)
8. Signal formatting — **fully specified** (section 5)
9. Telegram delivery — **tested and working**
10. Backtesting — *not started*
11. Paper-signal testing — *not started*
12. Further automation — *out of scope until 10–11 complete*

---

*This strategy is proprietary to Frenzo. Do not change the rules above without his explicit sign-off — this file documents his exact specification, not general SMC/ICT theory.*
```
