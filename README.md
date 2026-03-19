<p align="center">
  <img src="InsurO.png" width="200px" alt="InsureO Logo">
</p>

<h1 align="center">InsureO</h1>

<p align="center">
  <strong>AI-Powered Parametric Insurance for India's Gig Delivery Workers</strong>
</p>

<p align="center">
  <em>"Where Disruption Meets Protection, and Hustle Never Breaks"</em>
</p>

---

## The Problem

India has 50M+ gig delivery workers on platforms like Swiggy, Zomato, and Blinkit. They earn daily with zero financial safety net. External disruptions regularly destroy their income:

| Disruption | Real Impact |
|---|---|
| Traffic Jam | 4 deliveries/hour to 1 — Rs.120/hour lost |
| Heavy Rain | Work stops — Rs.300+ lost per event |
| Platform Algorithm Change | Orders drop 75% overnight, no explanation |
| Restaurant Delay | 30-min wait = 1 missed delivery = Rs.60-80 lost |

No insurance product in India currently protects gig workers from these real-time income shocks.

---

## What InsureO Does

InsureO is an AI-driven parametric insurance platform. It auto-detects disruptions, verifies actual earning loss, and triggers instant payouts — no manual claims, no paperwork.

The system does not ask workers to prove their loss. It detects it, verifies it, and pays it.

---

## Core Concept: Earning Efficiency Model

Instead of asking "Is there traffic?", InsureO asks "Has the worker's earning ability actually dropped?"

```
efficiency = current_earnings / expected_earnings
```

Payout triggers only when:
1. An external disruption is confirmed
2. AND worker efficiency drops below 0.5

| Metric | Normal | Disruption |
|---|---|---|
| Orders/hour | 4 | 1 |
| Earnings/hour | Rs.160 | Rs.40 |
| Efficiency | 1.0 | 0.25 — payout triggered |

---

## Cold Start — New Worker Onboarding

New workers have no history. InsureO solves this with a 3-stage baseline approach:

- Week 1-2: Use city-level averages for the worker's zone and time slot as a temporary baseline
- Week 3-4: Blend city average (60%) with the worker's own emerging data (40%)
- Week 5+: Worker's personal historical baseline takes full effect

During the cold-start period, payout thresholds are slightly stricter (efficiency must drop below 0.4 instead of 0.5) to offset the lower confidence in the baseline. This protects against early exploitation while still covering genuine disruptions.

---

## Disruptions Covered

### 1. Traffic Congestion

Detection:
```
delay_ratio = current_delivery_time / normal_delivery_time

IF delay_ratio > 1.5
AND traffic_level = HIGH
AND efficiency < 0.5
AND route_deviation < 1.3
-> DISRUPTION TRIGGERED
```

Route misuse prevention: The system compares the worker's GPS path against the optimal route available at delivery time. If the worker ignored a suggested route and took a provably heavier traffic path, the claim is reduced proportionally or flagged.

```
route_deviation = actual_route_time / optimal_route_time
IF route_deviation > 1.3 AND route_was_suggested = true
-> payout reduced or flagged
```

Payout example:

| Metric | Value |
|---|---|
| Expected Earnings | Rs.160/hour |
| Actual Earnings | Rs.40/hour |
| Loss | Rs.120 |
| Coverage | 70% |
| Final Payout | Rs.84 |

---

### 2. Weather Disruption

Detection:
```
IF rainfall > 100mm OR weather alert active
AND worker was active in affected zone
-> loss = hourly_income x hours_lost
-> payout = loss x 80% coverage
```

Predictive alert (advisory only):
```
Warning: Heavy rain expected at 7 PM
Predicted income drop: 25%
Suggestion: Work early hours or switch to Zone B
```

Alerts are advisory. A worker who ignores a suggestion and still suffers a genuine efficiency drop remains fully eligible for compensation. Alerts help workers earn more — they are not conditions for payout eligibility.

API fallback: If the Weather API is unavailable, the system falls back to last-known weather data cached within the past 30 minutes. If the outage exceeds 30 minutes, affected claims are held in a pending queue and processed once data is restored. Workers are never denied a payout due to a third-party API failure — claims are deferred, not rejected.

Payout example:

| Metric | Value |
|---|---|
| Rainfall | 130mm |
| Hours Lost | 3 hrs |
| Loss | Rs.300 |
| Coverage | 80% |
| Final Payout | Rs.240 |

---

### 3. Platform Algorithm Disruption

Platforms like Zomato and Swiggy update order allocation algorithms silently. Orders can drop 75% overnight — invisible income shock that no existing system detects.

Data source: InsureO does not rely on a formal platform API partnership. Detection is done through worker-side behavioral data — orders received, idle time, earnings per hour — which the worker shares by using the app. This is similar to how a fintech app infers spending patterns without bank API access.

Detection:
```python
z_score = (current_orders - historical_mean) / std_dev

IF z_score < -2
AND no weather or traffic event active
AND zone demand is normal (cross-worker check)
-> ALGORITHM DISRUPTION DETECTED
```

Advanced: Isolation Forest (scikit-learn) for behavioral anomaly detection across worker history.

| Metric | Value |
|---|---|
| Historical Avg Orders | 12/hour |
| Orders After Update | 3/hour |
| Drop | 75% — Flagged |

---

### 4. Restaurant Preparation Delay

Not every slow order is a disruption. InsureO uses a tolerance buffer so natural variation in prep time is never penalised.

Detection with buffer:
```
tolerance_threshold = avg_prep_time x 1.5   <- natural variation, no compensation
delay_threshold     = avg_prep_time x 2.0   <- confirmed delay, compensation begins

compensable_wait = current_prep_time - tolerance_threshold
```

If average prep time is 10 min:
- Up to 15 min — normal, no payout
- 15 to 20 min — partial compensation begins
- Beyond 20 min — full disruption, payout applies

Loss formula:
```
loss = (compensable_wait / avg_delivery_time) x earning_per_delivery
payout = loss x coverage_factor (70-80%)
```

| Metric | Value |
|---|---|
| Avg Prep Time | 10 min |
| Tolerance Buffer | 15 min |
| Current Prep Time | 30 min |
| Compensable Wait | 15 min |
| Loss | Rs.60 |
| Coverage | 75% |
| Final Payout | Rs.45 |

---

## Payout Model

```
Income Loss = Expected Earnings - Actual Earnings
Payout = Coverage % x Income Loss
```

Weekly premiums (aligned to gig worker pay cycles):

| Risk Zone | Premium |
|---|---|
| Low | Rs.20/week |
| Medium | Rs.25/week |
| High | Rs.30/week |

Coverage based on reliability score:

| Reliability Score | Coverage |
|---|---|
| 90-100 | 80% |
| 70-89 | 70% |
| Below 70 | 60% |

Financial sustainability: The Rs.20-30/week premium is intentionally set as a pilot-phase price point. Actuarial viability is maintained through three mechanisms — the efficiency threshold (0.5) filters out low-severity events, the tolerance buffer on restaurant delays reduces claim frequency, and the route deviation check removes fraudulent traffic claims. The business model in full scale depends on volume (large worker base, low individual payout frequency) and a reinsurance partnership where a licensed insurer underwrites the risk pool. InsureO operates as a technology and distribution layer, not as the direct risk-bearer. The current model is designed to demonstrate the detection engine and payout logic — full actuarial pricing will be done with an insurance partner at scale.

---

## Worker Performance Matrix

Workers are scored on their performance during normal days — no disruption, no bad weather, no traffic issues. This adds a genuineness dimension to the fraud check.

| Parameter | What It Measures |
|---|---|
| Avg orders/hour on normal days | Baseline productivity |
| Acceptance rate | Willingness to work |
| Active hours vs login hours | Actual working effort |
| On-time delivery rate | Reliability |
| Claim frequency vs disruption frequency | Abuse pattern detection |

How it affects payouts:

```
IF genuineness_score HIGH -> fast processing, coverage at 80%
IF genuineness_score LOW  -> manual review, coverage at 60%
```

Workers who are genuinely productive on normal days get faster, higher payouts when a real disruption hits. Workers with consistently low normal-day activity face more scrutiny before claims are approved.

---

## AI Risk Assessment

Every worker gets a daily risk score based on:
- Historical order patterns
- Live weather and traffic conditions
- Restaurant prep history in their zone
- Worker Performance Matrix data

| Output | Example Value |
|---|---|
| Earnings Score | 78 / 100 |
| Predicted Earnings Today | Rs.1,200 |
| Disruption Probability | 35% |

---

## Additional Features

**Predictive Traffic Avoidance** — predicts congestion 15 minutes ahead and alerts workers to reposition before getting stuck.

**Smart Zone Switching** — suggests zones with better demand and lower traffic in real time. Example: "Move to Zone B — 25% higher earning potential right now."

**Loss Heatmap** — live city map showing red (loss zones), yellow (moderate), and green (high earning) areas.

**Micro-Compensation Engine** — real-time credits every 15 minutes during confirmed disruptions instead of waiting for hourly payouts.

**Community Disruption Reporting** — workers can report accidents, restaurant issues, and platform glitches. Anti-abuse mechanism: a report only contributes to detection if at least 3 independent workers in the same zone report within a 15-minute window. Single or coordinated small-group reports are weighted lower. Repeated false reporters get their reporting trust score reduced, which also affects their claim genuineness score. This prevents coordinated fraud rings from gaming the community feature.

**Voice Alerts — Hands-Free Assistance** — all critical events are spoken aloud through the phone, so workers do not need to look at the screen while driving.

| Alert Type | Example |
|---|---|
| Risk | "Heavy traffic ahead. Expect delay." |
| Opportunity | "High demand nearby. Move for better earnings." |
| Guidance | "Waiting too long. Consider switching location." |
| System | "Disruption detected. Compensation initiated." |

Voice alert logic:
- Activates when GPS speed > 5 km/h (not 10) to cover workers moving slowly in traffic
- Workers fully stopped (speed = 0, engine idle) get a screen notification instead
- Max 1 alert per 2 minutes to avoid distraction
- Multi-language: English, Hindi, Odia

---

## Fraud Detection

| Type | Signals Checked |
|---|---|
| Traffic | GPS route + speed + route deviation + nearby worker comparison |
| Weather | Weather API + zone alerts + area-wide activity patterns |
| Algorithm | Cross-worker order distribution + login patterns |
| Restaurant | Timestamps + tolerance buffer + multi-worker corroboration |
| All | Worker Performance Matrix + genuineness score + claim frequency |

API failure policy: If a verification API (weather, traffic) is unavailable, the claim is queued — not rejected. Once data is restored, the queue is processed automatically. Workers are notified that their claim is pending, not denied.

---

## Go-To-Market

Pilot city: Bhubaneswar (manageable scale, strong two-wheeler delivery density, lower CAC than metros).

Target platform: Swiggy and Zomato delivery partners in Bhubaneswar as the first cohort.

First 1,000 workers: Partner with 2-3 delivery partner community groups and WhatsApp networks active in the city. Offer the first month free to build trust and collect real baseline data. Onboard through a simple app referral system where existing workers invite peers.

Underwriting: InsureO is the technology and distribution layer. A licensed IRDAI-registered insurance partner underwrites the actual risk pool. This is the same model used by platforms like Digit and Acko in their early embedded insurance products.

---

## System Architecture

```
+------------------------------------------+
|   External APIs (with fallback cache)    |
|   Weather, Traffic, Maps, Mock Delivery  |
+------------------------------------------+
                    |
+------------------------------------------+
|        Backend  (Node.js / FastAPI)      |
|   Event Detection, Parametric Triggers   |
|   Fraud Detection, Route Check, Queuing  |
+------------------------------------------+
        |                    |
+---------------+   +--------------------+
|  AI Engine    |   |  Real-Time Layer   |
|  Python       |   |  WebSockets        |
|  scikit-learn |   |  Socket.io         |
|  pandas       |   |  Map + Voice push  |
+---------------+   +--------------------+
        |                    |
+------------------------------------------+
|   Database                               |
|   PostgreSQL (structured data)           |
|   MongoDB (logs, alerts, claim queue)    |
+------------------------------------------+
                    |
        +-----------+------------+
        |                        |
+----------------+   +--------------------------+
| Razorpay       |   |  Mobile App              |
| Instant payout |   |  React, Dashboard, Map   |
|                |   |  TTS Engine, Voice       |
+----------------+   +--------------------------+
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, Leaflet / Google Maps API |
| Mobile | React Native |
| Backend | Node.js (Express) + Python (FastAPI) |
| AI / ML | scikit-learn, Isolation Forest, Z-score, Random Forest |
| Data Processing | pandas, numpy |
| Real-time | WebSockets (Socket.io) |
| Voice | Google TTS API / Expo Speech |
| Database | PostgreSQL + MongoDB |
| External APIs | OpenWeather, Google Maps Traffic, Mock Delivery API |
| Payments | Razorpay |

---

## End-to-End Workflow

```
1. Worker logs in -> cold-start or personal baseline loaded
2. Real-time monitoring starts (GPS, orders, traffic, weather)
3. Disruption signal detected
4. Efficiency score calculated — IF < 0.5, proceed
5. External cause validated via API (with fallback if API down)
6. Route deviation check (traffic claims)
7. Fraud + genuineness score check
8. Valid -> claim auto-triggered
9. Payout = coverage % x loss, adjusted by reliability score
10. Razorpay processes instant payment
11. Voice alert: "Compensation initiated."
```

---

## Impact

| Metric | Estimate |
|---|---|
| Target Users | 50M+ gig workers in India |
| Monthly Income Loss per Worker | Rs.2,000 to Rs.4,000 |
| Weekly Premium | Rs.20 to Rs.30 |
| Payout per Event | Rs.45 to Rs.240 |

InsureO turns gig worker insurance from a reactive product into a proactive earnings protection system.

---

## Development Roadmap

| Phase | Focus | Status |
|---|---|---|
| Phase 1 | Core setup, baseline tracking, efficiency model, weather + traffic APIs | In Progress |
| Phase 2 | Algorithm disruption detection, restaurant delay, fraud layer, databases | Planned |
| Phase 3 | Razorpay integration, auto-claim pipeline, reliability scoring, WebSockets | Planned |
| Phase 4 | Heatmap, zone switching, predictive alerts, micro-compensation, community reporting | Planned |
| Phase 5 | Voice alerts, multi-language, mobile UX polish | Planned |
| Phase 6 | End-to-end testing, fraud stress tests, beta with real workers, submission | Planned |

---

## Team

Built by **Aurelia Axis**

---

## License

This project was built for hackathon purposes. All rights reserved by the team.
