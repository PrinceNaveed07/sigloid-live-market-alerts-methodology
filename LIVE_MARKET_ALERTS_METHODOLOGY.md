# Sigloid Live Market Alerts — Technical Methodology Reference

This document provides the public technical methodology for Sigloid Live Market Alerts.

It explains how Sigloid defines completed daily market structure, detects bullish and bearish live structure breaches, records the Reference Price, calculates Deviation, assigns and updates the Risk Level, monitors active alerts, and closes invalidated alerts.

This is a simplified public reference. It does not disclose Sigloid’s complete production code, private constants, credentials, database design, server configuration, internal services, or proprietary infrastructure.

## Official Sigloid Pages

- [Live Market Alerts](https://sigloid.com/live-market-alerts)
- [User-facing Live Market Alerts methodology](https://sigloid.com/resources/methodology/live-market-alerts)
- [Sigloid](https://sigloid.com)

---

## 1. What Live Market Alerts Are

Sigloid Live Market Alerts identify assets whose live price has moved beyond a market-structure boundary calculated from completed daily candles.

A **bullish alert** means live price is trading above completed 55-day resistance.

A **bearish alert** means live price is trading below completed 55-day support.

The alerts are generated through rules-based market-structure detection. They are not discretionary analyst calls, price predictions, or automatic trade instructions.

---

## 2. Data Basis

The support and resistance levels used for alert detection are calculated from completed daily candles only.

The current incomplete daily candle is excluded from the completed 55-day structure calculation. Live price is then compared with those locked daily levels.

This creates two separate data layers:

1. **Completed daily structure** — the reference support and resistance levels.
2. **Current live price** — the price used to test whether those levels are being breached intraday.

---

## 3. Completed 55-Day Structure

For an asset with at least 55 completed daily candles, Sigloid calculates:

```text
55-day resistance =
maximum high of the latest 55 completed daily candles
```

```text
55-day support =
minimum low of the latest 55 completed daily candles
```

Simplified notation:

```text
resistance_55 = max(high[-55:])
support_55 = min(low[-55:])
```

An asset without sufficient completed daily history is not eligible for the standard 55-day calculation until enough data becomes available.

---

## 4. Bullish Alert Detection

A bullish Live Market Alert may be created when the latest eligible live price is above completed 55-day resistance.

```text
live_price > resistance_55
```

Simplified pseudocode:

```python
if live_price > completed_55_day_high:
    create_bullish_alert()
```

The alert means that upside structure has been breached intraday. It does not mean that the move has been confirmed by the daily close.

---

## 5. Bearish Alert Detection

A bearish Live Market Alert may be created when the latest eligible live price is below completed 55-day support.

```text
live_price < support_55
```

Simplified pseudocode:

```python
if live_price < completed_55_day_low:
    create_bearish_alert()
```

The alert means that downside structure has been breached intraday. It does not mean that the move has been confirmed by the daily close.

---

## 6. Live Alert vs Daily-Close Confirmation

Live Market Alerts are intraday structure events. They appear when live price moves beyond a level calculated from completed daily candles.

Daily-close confirmation is separate:

- A bullish structure break is confirmed only if the daily candle closes above the relevant resistance level.
- A bearish structure break is confirmed only if the daily candle closes below the relevant support level.

This distinction matters because an intraday breach can reverse before the daily candle closes.

---

## 7. Price Freshness

Sigloid checks whether the available live price is sufficiently recent before using it for alert detection.

If the preferred live-price source is unavailable or stale, the system may use another recent intraday market value. If no sufficiently reliable price is available, the asset is skipped until fresh data returns.

The public methodology does not disclose private freshness thresholds or internal source-priority settings.

---

## 8. Duplicate Alert Protection

Sigloid applies duplicate protection so the same active structure breach is not repeatedly published during consecutive scans.

The public logic includes:

- active-alert checks;
- symbol and direction checks;
- short-term cooldown handling; and
- recovery of stored active alerts after a restart.

Exact timing constants and production implementation details are not published.

---

## 9. Reference Price

**Reference Price** is the live market price recorded when Sigloid detects and creates the alert.

```text
reference_price = detected_live_price
```

Reference Price is not necessarily identical to the exact 55-day support or resistance level. Price may already have moved beyond the structure boundary when detection is processed.

Reference Price remains fixed for that alert and is used to measure subsequent movement.

---

## 10. Current Price

**Current Price** is the latest tracked market price displayed for an active alert.

It updates independently from the stored Reference Price. This allows users to compare:

- where the alert was detected; and
- where the asset is trading now.

---

## 11. Deviation

**Deviation** measures how far Current Price has moved from Reference Price in the direction of the alert.

### Bullish Deviation

```text
((current_price - reference_price) / reference_price) × 100
```

### Bearish Deviation

```text
((reference_price - current_price) / reference_price) × 100
```

Interpretation:

- A positive value means price has moved in the alert direction.
- A negative value means price has moved against the alert direction.
- A large positive value means price has already moved materially away from the Reference Price and should be reviewed on the live chart before any decision is made.

---

## 12. Alert Bias

**Alert Bias** describes the direction of the detected structure breach.

```text
Bullish = live price above completed 55-day resistance
Bearish = live price below completed 55-day support
```

Alert Bias describes the event direction only. It does not guarantee trend continuation and is not a personal recommendation.

---

## 13. Initial Risk Level

**Risk Level** is the active invalidation level assigned to an alert.

When an alert is created, Sigloid calculates the initial Risk Level from short-term live moving-average context.

That live context combines:

- completed daily closing prices; and
- current live price as a provisional current-day value.

For a bullish alert, the initial Risk Level is below live price.

For a bearish alert, the initial Risk Level is above live price.

The production system applies a small private buffer around the risk area. The exact constant is not published.

---

## 14. Live Moving-Average Context

Selected live moving averages are calculated by combining completed daily closes with current live price.

Simplified construction:

```text
synthetic_close_series =
completed_daily_closes + current_live_price
```

The live moving average is then calculated from the required number of values in that provisional series.

```text
live_MA(period) =
average of the latest period values,
including current live price
```

This allows the Risk Level framework to respond during the current day rather than waiting for the next completed daily candle.

---

## 15. Risk Level Updates

Sigloid reviews active Risk Levels through scheduled structure checks.

The simplified public framework has three broad phases.

### Phase 1 — Early Alert

The Risk Level follows short-term live moving-average context.

### Phase 2 — Established Alert

If price continues in the alert direction and short-term structure improves, the Risk Level may move toward:

- Reference Price;
- breakeven context; or
- medium-term moving-average context.

### Phase 3 — Mature Structure

Once the alert becomes more established, shorter-term completed structure may be used to maintain the invalidation area.

For bullish alerts, downside structure is relevant.

For bearish alerts, upside structure is relevant.

Exact thresholds, buffers, timing rules, and transition conditions remain private.

---

## 16. Simplified Risk Level Pseudocode

```python
if alert_is_new:
    risk_level = short_term_live_structure

elif alert_is_established:
    risk_level = breakeven_or_medium_term_context

elif alert_is_mature:
    risk_level = bias_aligned_shorter_term_structure
```

Directional validity:

```text
Bullish alert: risk_level < current_price
Bearish alert: risk_level > current_price
```

---

## 17. Alert Closure

An active alert closes when live price reaches or crosses the active Risk Level.

### Bullish Alert

```text
current_price <= risk_level
```

### Bearish Alert

```text
current_price >= risk_level
```

After invalidation:

- the alert is closed;
- it is removed from the active Live Market Alerts page; and
- its completed outcome can be retained for historical research.

---

## 18. Alert Lifecycle

The simplified Alert Lifecycle is:

```text
Completed daily structure calculated
→ live price breaches 55-day structure
→ alert created
→ Reference Price recorded
→ initial Risk Level assigned
→ Current Price and Deviation monitored
→ Risk Level reviewed and updated
→ alert remains active or is invalidated
→ completed outcome retained for research
```

---

## 19. Re-entry Logic

A previously closed structure may become relevant again if price first pulls back and later breaches the completed 55-day level again.

Simplified sequence:

```text
initial alert created
→ price pulls back through required short-term structure
→ pullback condition recorded
→ price breaches the 55-day level again
→ a new alert may be created
```

Re-entry logic uses additional pullback, direction, duplicate-protection, and eligibility checks. Exact production rules and timing conditions are not published.

---

## 20. Position Size Formula

The position-size calculator uses a user-defined monetary risk amount and the absolute distance between Current Price and Risk Level.

```text
position_size =
risk_amount / abs(current_price - risk_level)
```

Simplified pseudocode:

```python
distance = abs(current_price - risk_level)
position_size = risk_amount / distance
```

This formula estimates quantity only. It does not automatically account for:

- leverage;
- trading fees;
- slippage;
- funding;
- exchange limits;
- liquidation risk;
- contract multipliers; or
- asset-specific execution constraints.

Users must independently validate the result before placing any trade.

---

## 21. Market Context at Alert Creation

When a new alert is created, Sigloid may record a research snapshot associated with the alert.

The context can include selected measures such as:

- live moving-average distances;
- momentum indicators;
- volatility measures;
- Bollinger Band width;
- ATR percentage;
- funding;
- open interest;
- long/short positioning;
- broader market bias; and
- relationship measures against major market benchmarks.

This makes it possible to study the market conditions that existed when the alert was detected.

The exact internal data model is not published.

---

## 22. Active Alert Monitoring

While an alert remains active, Sigloid can monitor and retain selected information including:

- Current Price;
- active Risk Level;
- favorable movement;
- adverse movement;
- alert age;
- live market context; and
- selected structure changes.

Periodic snapshots may be recorded so an alert can be studied across its lifecycle rather than only at creation and closure.

---

## 23. Maximum Favorable and Adverse Excursion

Sigloid tracks the largest favorable and adverse movement reached while an alert remains active.

### Maximum Favorable Excursion

**Maximum Favorable Excursion (MFE)** measures the largest move in the alert direction.

For bullish alerts:

```text
MFE =
((highest_price_since_alert - reference_price) / reference_price) × 100
```

For bearish alerts:

```text
MFE =
((reference_price - lowest_price_since_alert) / reference_price) × 100
```

### Maximum Adverse Excursion

**Maximum Adverse Excursion (MAE)** measures the largest move against the alert direction.

For bullish alerts:

```text
MAE =
((reference_price - lowest_price_since_alert) / reference_price) × 100
```

For bearish alerts:

```text
MAE =
((highest_price_since_alert - reference_price) / reference_price) × 100
```

MFE and MAE are historical research measurements. They are not forecasts or trade recommendations.

---

## 24. Alert Outcomes

When an alert closes, Sigloid may retain an outcome record for research.

A completed record can include:

- Reference Price;
- exit price;
- Alert Bias;
- active duration;
- final Risk Level context;
- percentage outcome;
- MFE;
- MAE;
- result classification; and
- selected market context.

This supports later analysis of how alerts behaved under different market conditions.

---

## 25. Notifications

After a new Live Market Alert is successfully created, Sigloid sends the alert to its public Telegram group and directs users to the Live Market Alerts page.

Notifications are intended to reduce manual scanning. Risk Level updates, periodic snapshots, and alert closures are not sent as separate public Telegram alerts.

---

## 26. Live Page Updates

The Live Market Alerts page receives active alert changes through a live data connection.

Three broad public update types describe the lifecycle:

```text
ENTRY
SL_UPDATE
EXIT
```

### ENTRY

Adds a newly created alert to the active page.

### SL_UPDATE

Updates the active Risk Level displayed on the alert card.

### EXIT

Removes the alert from the active page after invalidation or closure.

These public labels describe visible alert changes. They do not document Sigloid’s internal event-processing architecture.

---

## 27. Search and Filters

The Live Market Alerts page allows users to:

- search by symbol or asset name;
- filter bullish alerts;
- filter bearish alerts;
- view all active alerts; and
- review alerts by displayed Deviation.

These controls affect how active alerts are displayed. They do not change the underlying detection methodology.

---

## 28. Chart and Asset Analysis Links

Each alert can provide access to:

- a live chart;
- the related Sigloid asset analysis page;
- the position-size calculator; and
- watchlist controls.

The live chart is intended for visual confirmation.

The asset analysis page provides additional closed-candle context, including structure, indicators, derivatives data, volatility, and market relationships.

---

## 29. Sigloid Index Context

The Live Market Alerts page also displays the current Sigloid Index bias as broader crypto market context.

The Index does not change the core 55-day alert-detection rule. It helps users interpret whether an alert is occurring within a broader trending market or a more sideways environment.

As a general research principle:

- in a broad trending regime, strongly linked assets may deserve more attention;
- in a sideways regime, weakly linked or asset-specific structure changes may carry more independent information.

This is contextual interpretation, not an additional alert-trigger condition.

---

## 30. Simplified End-to-End Pseudocode

```python
for symbol in eligible_symbols:

    completed_structure = load_completed_daily_structure(symbol)
    live_price = load_fresh_live_price(symbol)

    if completed_structure is unavailable:
        continue

    if live_price is unavailable:
        continue

    resistance_55 = max(last_55_completed_daily_highs)
    support_55 = min(last_55_completed_daily_lows)

    if live_price > resistance_55:
        if no_active_bullish_alert(symbol):
            reference_price = live_price
            risk_level = calculate_initial_bullish_risk_level()

            create_bullish_alert(
                symbol=symbol,
                reference_price=reference_price,
                risk_level=risk_level,
            )

    elif live_price < support_55:
        if no_active_bearish_alert(symbol):
            reference_price = live_price
            risk_level = calculate_initial_bearish_risk_level()

            create_bearish_alert(
                symbol=symbol,
                reference_price=reference_price,
                risk_level=risk_level,
            )
```

Simplified active-alert monitoring:

```python
for alert in active_alerts:

    current_price = load_current_price(alert.symbol)
    updated_risk_level = review_risk_level(alert)

    if updated_risk_level is valid:
        alert.risk_level = updated_risk_level

    if alert.bias == "BULLISH":
        if current_price <= alert.risk_level:
            close_alert(alert)

    elif alert.bias == "BEARISH":
        if current_price >= alert.risk_level:
            close_alert(alert)
```

The pseudocode is explanatory only. It is not Sigloid’s complete production implementation.

---

## 31. What This Public Reference Does Not Publish

This repository intentionally does not publish:

- complete production source code;
- exact private buffers or thresholds;
- exact cooldown timings;
- private price-freshness limits;
- full Risk Level transition rules;
- production database schemas;
- internal queues or event schemas;
- server paths;
- service names;
- API keys;
- credentials;
- environment variables;
- notification tokens;
- infrastructure configuration; or
- proprietary monitoring and recovery logic.

The purpose is to provide technical transparency without exposing the complete production system.

---

## 32. Methodology Limitations

Live Market Alerts have important limitations.

### Intraday Breaches Can Fail

Price can move beyond completed 55-day structure and reverse before the daily close.

### Fast Markets Can Create Price Gaps

Reference Price may differ from the exact support or resistance boundary.

### Data Can Be Delayed or Unavailable

An asset may be skipped when live data is stale, incomplete, or unavailable.

### Risk Level Is Model-Derived

Risk Level is not a personalized stop loss. It does not account for an individual user’s capital, leverage, liquidity, execution method, account restrictions, or tolerance for loss.

### Historical Outcomes Do Not Guarantee Future Results

Past alert behavior does not prove that similar future alerts will behave in the same way.

### Market Conditions Change

Volatility, liquidity, funding, correlations, and market regime can change after an alert is created.

---

## 33. Research-Only Use

Sigloid Live Market Alerts are provided as a market-research and scanning layer.

They are not:

- financial advice;
- investment recommendations;
- buy or sell instructions;
- guaranteed trading opportunities;
- price predictions;
- portfolio instructions; or
- personalized risk advice.

Users must apply their own confirmation rules, risk limits, position sizing, execution plan, and independent judgment.

---

## 34. Methodology Updates

Sigloid may update this public methodology as the platform evolves.

Material updates may include:

- revised public field definitions;
- improved formulas;
- clearer pseudocode;
- new alert-card fields;
- additional outcome measurements; or
- updated explanations of the Alert Lifecycle.

Changes to this document do not necessarily disclose every change made to the production implementation.

---

## 35. Current Public Version

```text
Methodology: Sigloid Live Market Alerts
Reference type: Public technical methodology
Status: Active
Last reviewed: July 2026
```

For the latest user-facing explanation, visit:

- [Live Market Alerts methodology](https://sigloid.com/resources/methodology/live-market-alerts)

For active alerts, visit:

- [Sigloid Live Market Alerts](https://sigloid.com/live-market-alerts)
