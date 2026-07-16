# Sigloid Live Market Alerts — Technical Methodology Reference

This document explains the public technical methodology behind Sigloid Live Market Alerts.

It describes how Sigloid calculates completed daily structure, detects live bullish and bearish alerts, assigns the Reference Price, calculates Deviation, manages Risk Level, and closes active alerts.

This is a simplified public reference. It does not publish Sigloid’s production bot, private constants, infrastructure, credentials, database design, or full proprietary implementation.

## Official Sigloid Pages

- Live Market Alerts: https://sigloid.com/live-market-alerts
- User-facing methodology: https://sigloid.com/resources/methodology/live-market-alerts
- Sigloid: https://sigloid.com

---

## 1. What Live Market Alerts Are

Sigloid Live Market Alerts identify assets whose live price has moved beyond a completed daily market-structure boundary.

A bullish alert means live price is trading above completed 55-day resistance.

A bearish alert means live price is trading below completed 55-day support.

The alerts are generated from rules-based price structure. They are not discretionary analyst calls, predictions, or automatic trade instructions.

---

## 2. Data Basis

The structure levels used for alert detection are calculated from completed daily candles only.

The current incomplete daily candle is excluded from the 55-day structure calculation.

Live price is then compared with those completed daily levels.

This creates two separate data layers:

## 1. completed daily structure; and

## 2. current live market price.

---

## 3. Completed 55-Day Structure

For assets with at least 55 completed daily candles, Sigloid calculates:

```text
55-day resistance =
maximum high of the latest 55 completed daily candles
```

```text
55-day support =
minimum low of the latest 55 completed daily candles
```

In simplified mathematical form:

```text
resistance_55 = max(high[-55:])
support_55 = min(low[-55:])
```

Assets without sufficient completed daily history are not eligible for the standard 55-day alert calculation.

## 4. Bullish Alert Detection

A bullish Live Market Alert is created when the latest tracked live price is above completed 55-day resistance.

live_price > resistance_55

Simplified pseudocode:

if live_price > completed_55_day_high:
    create_bullish_alert()

The alert indicates that upside structure has been breached intraday.

It does not mean the move has been confirmed by the daily close.

## 5. Bearish Alert Detection

A bearish Live Market Alert is created when the latest tracked live price is below completed 55-day support.

live_price < support_55

Simplified pseudocode:

if live_price < completed_55_day_low:
    create_bearish_alert()

The alert indicates that downside structure has been breached intraday.

It does not mean the move has been confirmed by the daily close.

## 6. Live Alert vs Daily-Close Confirmation

Live Market Alerts are intraday structure events.

They appear when live price moves beyond a completed daily level.

Daily confirmation is separate.

A bullish structure break is confirmed only if the daily candle later closes above the relevant resistance level.

A bearish structure break is confirmed only if the daily candle later closes below the relevant support level.

This distinction matters because intraday moves can reverse before the daily close.

## 7. Price Freshness

Sigloid checks whether the available live price is recent enough before using it for alert detection.

If the preferred live-price source is unavailable or stale, the system may use a recent intraday market value.

If no sufficiently reliable live price is available, the asset is skipped until fresh data returns.

The public methodology does not expose private freshness thresholds or internal source-priority settings.

## 8. Duplicate Alert Protection

Sigloid prevents the same active bullish or bearish structure from being repeatedly added during each scan.

Duplicate protection is applied through:

active-alert checks;
symbol and direction checks;
short-term cooldown handling; and
restart recovery from stored active alerts.

The exact production implementation and timing constants are not published.

## 9. Reference Price

Reference Price is the live market price at which Sigloid detected and created the alert.

It is the alert’s recorded starting price.

Reference Price is not necessarily identical to the exact 55-day support or resistance level because live price may already have moved beyond that level when detection is processed.

reference_price = detected_live_price

Reference Price is used to measure how the alert behaves after detection.

## 10. Current Price

Current Price is the latest tracked market price displayed for the active alert.

It updates independently from the stored Reference Price.

This allows users to compare:

where the alert was detected; and
where the asset is trading now.

## 11. Deviation

Deviation measures how far Current Price has moved from Reference Price in the direction of the alert.

### Bullish deviation
((current_price - reference_price) / reference_price) × 100

### Bearish deviation
((reference_price - current_price) / reference_price) × 100

A positive deviation means price is moving in the alert direction.

A negative deviation means price has moved against the alert direction.

A large positive deviation means price has already moved significantly from the Reference Price and should be reviewed carefully on the chart before any decision is made.

## 12. Alert Bias

Alert Bias describes the direction of the detected structure change.

Bullish = live price above completed 55-day resistance
Bearish = live price below completed 55-day support

Alert Bias describes the event direction only.

It does not represent a guaranteed trend continuation or a personal recommendation.

## 13. Initial Risk Level

Risk Level is the active invalidation level assigned to an alert.

When an alert is created, Sigloid calculates the initial Risk Level from short-term live moving-average context.

Live moving-average context combines:

completed daily close history; and
the current live price as a provisional current-day value.

For a bullish alert, the initial invalidation level is below live price.

For a bearish alert, the initial invalidation level is above live price.

The production system applies a small private risk buffer. The exact constant is not published.

## 14. Live Moving-Average Context

Sigloid calculates selected live moving averages by combining completed daily closes with the current live price.

Simplified example:

synthetic_close_series =
completed_daily_closes + current_live_price

The live moving average is then calculated from that provisional series.

live_MA(period) =
average of the latest period values,
including the current live price

This allows the Risk Level framework to respond during the current day rather than waiting for the next daily close.

## 15. Risk Level Updates

Sigloid reviews active Risk Levels through scheduled structure checks.

The public logic has three broad phases.

### Phase 1 — Early alert

The Risk Level follows short-term live moving-average context.

### Phase 2 — Established alert

If price continues in the alert direction and short-term structure improves, the Risk Level moves toward:

the Reference Price;
breakeven context; or
medium-term moving-average context.

### Phase 3 — Mature structure

Once the alert becomes more established, Sigloid can use shorter-term completed structure levels to maintain the invalidation area.

For bullish alerts, downside structure is used.

For bearish alerts, upside structure is used.

Exact thresholds, buffers, and transition conditions remain private.

## 16. Simplified Risk Level Pseudocode
if alert_is_new:
    risk_level = short_term_live_structure

elif alert_is_established:
    risk_level = breakeven_or_medium_term_context

elif alert_is_mature:
    risk_level = bias_aligned_shorter_term_structure

For bullish alerts:

risk_level must remain below current_price

For bearish alerts:

risk_level must remain above current_price

## 17. Alert Closure

An active alert is closed when live price reaches or crosses the active Risk Level.

### Bullish alert
current_price <= risk_level

### Bearish alert
current_price >= risk_level

When this happens:

the alert is treated as invalidated;
it is removed from the active Live Market Alerts page; and
its outcome is retained for research and historical analysis.

## 18. Alert Lifecycle

The simplified Alert Lifecycle is:

Completed daily structure calculated
→ live price breaches 55-day structure
→ alert created
→ Reference Price recorded
→ initial Risk Level assigned
→ Current Price and Deviation monitored
→ Risk Level updated
→ alert invalidated or remains active
→ closed outcome retained for research

## 19. Re-entry Logic

A previously invalidated or closed structure can become relevant again if price first pulls back and later breaches the completed 55-day level again.

A simplified re-entry process is:

initial alert created
→ price pulls back through short-term structure
→ pullback condition recorded
→ price breaches the 55-day level again
→ new alert may be created

Re-entry detection uses additional direction, pullback, and duplicate-protection checks.

The exact production rules and timing conditions are not published.

## 20. Position Size Formula

The position-size calculator uses a user-defined risk amount and the distance between Current Price and Risk Level.

position_size =
risk_amount / abs(current_price - risk_level)

Simplified example:

distance = abs(current_price - risk_level)
position_size = risk_amount / distance

This formula calculates quantity only.

It does not account for:

leverage;
fees;
slippage;
funding;
exchange limits;
liquidation risk;
contract multipliers; or
asset-specific execution constraints.

Users remain responsible for validating the result before placing any trade.

## 21. Market Context Stored at Alert Creation

When a new alert is created, Sigloid records a market snapshot associated with that alert.

The stored research context can include selected measures such as:

live moving-average distances;
momentum indicators;
volatility measures;
Bollinger Band width;
ATR percentage;
funding;
open interest;
long/short positioning;
broader market bias; and
relationship measures against major market benchmarks.

This allows later analysis to examine the market conditions that existed when the alert was detected.

The exact internal data schema is not published.

## 22. Active Alert Monitoring

While an alert remains active, Sigloid monitors:

Current Price;
active Risk Level;
favorable movement;
adverse movement;
alert age;
live market context; and
selected structure changes.

Periodic snapshots may be recorded so the alert can be analyzed over time rather than only at entry and exit.

## 23. Maximum Favorable and Adverse Movement

Sigloid tracks the best and worst movement reached while an alert remains active.

### Maximum Favorable Excursion

Maximum Favorable Excursion measures the largest move in the alert direction.

For bullish alerts:

MFE =
(highest_price_since_alert - reference_price)
/
reference_price × 100

For bearish alerts:

MFE =
(reference_price - lowest_price_since_alert)
/
reference_price × 100

### Maximum Adverse Excursion

Maximum Adverse Excursion measures the largest move against the alert direction.

For bullish alerts:

MAE =
(reference_price - lowest_price_since_alert)
/
reference_price × 100

For bearish alerts:

MAE =
(highest_price_since_alert - reference_price)
/
reference_price × 100

MFE and MAE are research measurements. They are not forecasts or trade recommendations.

## 24. Alert Outcomes

When an alert closes, Sigloid retains the outcome for research.

A closed alert record can include:

Reference Price;
exit price;
Alert Bias;
active duration;
final Risk Level context;
percentage outcome;
MFE;
MAE;
result classification; and
selected market context.

This supports later analysis of how different market structures performed under different conditions.

## 25. Notifications

When a new Live Market Alert is successfully created, Sigloid sends a notification through its configured notification channels.

The public Telegram notification directs users to the Live Market Alerts page.

Notifications are intended to reduce manual market scanning.

Risk Level updates, snapshots, and alert closures are not necessarily sent as separate public notifications.

## 26. Live Page Updates

The Live Market Alerts page receives active alert updates through a live data connection.

The page can receive three broad update types:

### ENTRY

### SL_UPDATE

### EXIT

### ENTRY

Adds a newly created alert to the live page.

### SL_UPDATE

Updates the active Risk Level shown on the alert card.

### EXIT

Removes the alert from the active page after invalidation or closure.

These labels describe the public alert lifecycle and do not expose Sigloid’s internal event-processing architecture.

## 27. Search and Filters

The Live Market Alerts page allows users to:

search by symbol or asset name;
filter bullish alerts;
filter bearish alerts;
view all active alerts; and
review alerts with positive Deviation.

These controls change how active alerts are displayed.

They do not change the underlying detection methodology.

## 28. Chart and Asset Analysis Links

Each alert provides access to:

the live chart;
the related Sigloid asset analysis page;
the position-size calculator; and
watchlist controls.

The chart is intended for live visual confirmation.

The asset analysis page provides closed-candle context, including structure, indicators, derivatives data, volatility, and market relationships.

## 29. Sigloid Index Context

The Live Market Alerts page also displays the current Sigloid Index bias.

The Sigloid Index provides broader crypto market context.

A market-wide bullish or bearish regime can make strongly linked asset moves more relevant.

A sideways market can make weakly linked or asset-specific structure changes more important.

The Sigloid Index does not change the basic 55-day alert detection rule.

It provides context for interpretation.

## 30. Simplified End-to-End Pseudocode
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
                risk_level=risk_level
            )

    elif live_price < support_55:
        if no_active_bearish_alert(symbol):
            reference_price = live_price
            risk_level = calculate_initial_bearish_risk_level()
            create_bearish_alert(
                symbol=symbol,
                reference_price=reference_price,
                risk_level=risk_level
            )

Active alert monitoring:

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

## 31. What This Public Reference Does Not Publish

This repository intentionally does not publish:

the full production strategy source code;
exact private buffer constants;
exact cooldown timings;
private price-freshness thresholds;
full Risk Level transition conditions;
production database schemas;
internal event queues;
server paths;
service names;
API keys;
credentials;
environment variables;
notification tokens;
infrastructure configuration; or
proprietary monitoring and recovery logic.

The purpose is technical transparency without exposing the complete production system.

## 32. Methodology Limitations

Live Market Alerts have important limitations.

### Intraday breaches can fail

Price can move beyond completed 55-day structure and reverse before the daily close.

### Fast markets can create price gaps

Reference Price may differ from the exact support or resistance boundary.

### Data can be delayed or unavailable

An asset may be skipped when live data is stale, incomplete, or unavailable.

### Risk Level is model-derived

Risk Level is not a personal stop loss and does not account for an individual user’s capital, leverage, liquidity, execution method, or tolerance for loss.

### Historical outcomes do not guarantee future results

Past alert behavior does not prove that similar future alerts will perform the same way.

### Market conditions change

Volatility, liquidity, funding, correlations, and market regime can change after an alert is created.

## 33. Research-Only Use

Sigloid Live Market Alerts are provided as a market research and scanning layer.

They are not:

financial advice;
investment recommendations;
buy or sell signals;
guaranteed trading opportunities;
price predictions;
portfolio instructions; or
personalized risk advice.

Users must apply their own confirmation rules, risk limits, position sizing, execution plan, and independent judgment.

## 34. Methodology Updates

Sigloid may update this public methodology as the platform changes.

Material changes may include:

revised public field definitions;
improved formulas;
clearer pseudocode;
new alert-card fields;
additional outcome measures; or
updated explanations of the Alert Lifecycle.

Changes to this document do not necessarily disclose every production implementation change.

## 35. Current Public Version
Methodology: Sigloid Live Market Alerts
Reference type: Public technical methodology
Status: Active
Last reviewed: July 2026

For the latest user-facing explanation, visit:

https://sigloid.com/resources/methodology/live-market-alerts

For active alerts, visit:

https://sigloid.com/live-market-alerts
