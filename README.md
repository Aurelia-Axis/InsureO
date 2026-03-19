<p align="center">
  <img src="InsurO.png" width="600px" alt="InsureO Logo">
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

New workers have no personal history. InsureO uses a 3-stage blended baseline:

- Week 1-2: City-level averages for the worker's zone and time slot used as temporary baseline
- Week 3-4: Blended — 60% city average, 40% worker's own emerging data
- Week 5+: Worker's personal historical baseline takes full effect

During cold start, the payout threshold is slightly stricter (efficiency must drop below 0.4 instead of 0.5) to offset lower confidence in the baseline. Genuine disruptions are still covered.

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

Route snapshot integrity: At the moment a delivery is assigned, the system calls the Maps API and stores an immutable snapshot of the optimal route — origin, destination, estimated time, and path — with a dispatch timestamp in the database. This snapshot is frozen and cannot be modified after dispatch. The fraud check always compares the worker's actual GPS path against this frozen snapshot, not a live recalculation. This prevents any retroactive comparison against a different route.

```
-- stored at dispatch time, immutable
route_snapshot = {
  delivery_id,
  origin,
  destination,
  optimal_path,
  estimated_time,
  dispatched_at   <- frozen timestamp
}

route_deviation = actual_route_time / snapshot.estimated_time
IF route_deviation > 1.3 AND snapshot.route_was_suggested = true
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

API fallback: If the Weather API goes down, the system uses the last cached data (valid for 30 minutes). If the outage exceeds 30 minutes, affected claims enter a pending queue and are processed automatically once the API recovers. Workers are notified their claim is pending — never that it is denied due to a system issue.

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

Platforms like Zomato and Swiggy silently update order allocation algorithms. Orders can drop 75% overnight with no explanation — invisible income shock.

Data source: InsureO does not require a formal platform API partnership. Detection uses worker-side behavioral data — orders received, idle time, earnings per hour — shared through the app. This follows the same principle as fintech apps that infer spending patterns without direct bank API access.

Detection:
```python
z_score = (current_orders - historical_mean) / std_dev

IF z_score < -2
AND no weather or traffic event active
AND demand_signal = GENUINE_DROP  <- see below
-> ALGORITHM DISRUPTION DETECTED
```

Platform-wide vs genuine demand drop: A cross-worker check alone is not enough. If Zomato penalises an entire zone simultaneously, all workers show the same drop — and a naive cross-worker check would mark it as normal. InsureO handles this with a two-layer demand signal:

- Layer 1 — Cross-platform check: Compare order volume across multiple platforms in the same zone (Swiggy vs Zomato vs Blinkit). If only one platform shows a zone-wide drop while others remain normal, it confirms a platform-specific algorithm change rather than a genuine demand drop.
- Layer 2 — Consumer demand proxy: Cross-reference with restaurant activity data (order timestamps from the restaurant side) and zone foot traffic patterns. If restaurants are receiving orders but workers on one platform are not getting assigned, the supply-side algorithm change is confirmed.

```
IF platform_A_drop = HIGH
AND platform_B_drop = NORMAL (same zone, same time)
AND restaurant_activity = NORMAL
-> PLATFORM ALGORITHM DISRUPTION confirmed
```

Isolation Forest training: The model is pre-trained on synthetic data generated from realistic gig worker patterns (order frequency distributions, time-of-day curves, zone-level demand variance). In the pilot phase with limited real workers, synthetic data fills the gap. As real worker data accumulates, the model is retrained in weekly batches. A minimum of 50 workers in a zone is required before the cross-worker validation layer is considered statistically reliable. Below that threshold, the system falls back to individual Z-score detection only, with a slightly higher anomaly threshold to compensate for reduced confidence.

| Metric | Value |
|---|---|
| Historical Avg Orders | 12/hour |
| Orders After Update | 3/hour |
| Drop | 75% — Flagged |

---

### 4. Restaurant Preparation Delay

Not every slow order is a disruption. InsureO uses a tolerance buffer so natural variation in prep time is not penalised.

```
tolerance_threshold = avg_prep_time x 1.5   <- natural variation, no compensation
delay_threshold     = avg_prep_time x 2.0   <- confirmed delay, compensation begins

compensable_wait = current_prep_time - tolerance_threshold
```

If average prep time is 10 min:
- Up to 15 min — normal, no payout
- 15 to 20 min — partial compensation begins
- Beyond 20 min — full disruption, payout applies

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

Weekly premiums:

| Risk Zone | Premium |
|---|---|
| Low | Rs.20/week |
| Medium | Rs.25/week |
| High | Rs.30/week |

Premium collection: Premiums are collected via UPI AutoPay (recurring mandate) set up at onboarding. The mandate auto-debits weekly on the worker's designated pay day. If a payment fails, the worker gets a 48-hour grace period with one retry. If premium is not paid within the grace period, coverage is paused — not cancelled. Any disruption that occurs during a lapsed coverage window is not eligible for payout. The worker can reinstate coverage immediately by clearing the due amount, and coverage resumes from the next shift after payment.

Coverage based on reliability score:

| Reliability Score | Coverage |
|---|---|
| 90-100 | 80% |
| 70-89 | 70% |
| Below 70 | 60% |

Financial sustainability: The Rs.20-30/week premium is a pilot-phase price point. Actuarial viability is maintained through the efficiency threshold (0.5), the tolerance buffer on restaurant delays, and the route deviation check — these together significantly reduce claim frequency. At full scale, the model depends on a reinsurance partnership where a licensed IRDAI insurer underwrites the risk pool. InsureO is the technology and distribution layer, not the direct risk-bearer. Full actuarial pricing will be set with the insurance partner based on real pilot claim data.

IRDAI and legal framework: During the pilot phase, real money moves via Razorpay but InsureO operates as a fintech payout platform, not as a licensed insurer. Payouts in the pilot are structured as income protection disbursements under a tech service agreement, not as insurance claims. This keeps the pilot outside IRDAI's insurance classification while still delivering real value to workers. Before commercial launch, InsureO will either register as an insurance aggregator under IRDAI regulations or operate under a white-label arrangement with a licensed insurer. The sandbox regulatory environment under IRDAI's Bima Sugam initiative is a potential path for formalising the pilot.

---

## Worker Performance Matrix

Workers are scored on their behaviour during normal days — no disruption, no bad weather, no traffic issues. This adds a genuineness dimension to fraud checks.

| Parameter | What It Measures |
|---|---|
| Avg orders/hour on normal days | Baseline productivity |
| Active hours vs login hours | Actual working effort |
| On-time delivery rate | Reliability |
| Claim frequency vs disruption frequency | Abuse pattern detection |

Note on fairness: The genuineness score does not penalise low intensity on its own. A part-time worker, someone recovering from illness, or a worker who selectively accepts orders for safety reasons will not be treated the same as an inactive worker gaming the system. The score is relative — it compares a worker's own current behaviour against their own past behaviour, not against other workers. A worker who consistently works 3 hours a day and has a steady pattern scores just as high as a full-time worker, as long as their behaviour is consistent. What the score flags is sudden unexplained inactivity combined with a spike in claims — not simply low volume.

```
genuineness_score = consistency of own pattern over time
                    NOT a comparison against other workers

IF claim_frequency spikes while normal_day_activity drops
-> flagged for review

IF pattern is consistent (even at low volume)
-> no penalty applied
```

---

## Worker Data Privacy and Consent

InsureO collects GPS location, order history, speed, idle time, and earnings data continuously during active shifts. This data is the foundation of the detection and payout system. The following framework governs how it is handled:

Data ownership: The worker owns their data. InsureO holds it under a data processing agreement, not as an owner. Workers can request a full export or deletion of their data at any time.

Consent at onboarding: Workers explicitly consent to each data category at sign-up — GPS tracking, earnings data, order history, and activity logs — through a plain-language consent screen in their preferred language. Consent is granular; workers can accept or decline individual categories.

Opt-out and impact: A worker can opt out of specific tracking. The tradeoff is transparent:
- Opting out of GPS: traffic and restaurant delay claims cannot be verified — those disruption types are excluded from coverage, but weather and algorithm disruption coverage remains active.
- Opting out of order history sharing: algorithm disruption detection is disabled for that worker.
- Opting out of earnings data: the efficiency model cannot run — full coverage is paused until consent is reinstated.

Workers are never forced to share data. The coverage scope adjusts to match what they consent to.

Data minimisation: Only data directly required for disruption detection and payout calculation is collected. No data is sold or shared with third parties, including the platforms (Zomato, Swiggy) the worker operates on.

Storage: All personal data is stored encrypted at rest. GPS data is retained for 90 days for claim audit purposes, then auto-deleted. Aggregate anonymised data (zone-level patterns) is retained for model training.

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

**Predictive Traffic Avoidance** — predicts congestion 15 minutes ahead so workers can reposition before getting stuck.

**Smart Zone Switching** — real-time zone suggestions based on demand and traffic. Example: "Move to Zone B — 25% higher earning potential right now."

**Loss Heatmap** — live city map: red (loss zones), yellow (moderate), green (high earning).

**Micro-Compensation Engine** — credits every 15 minutes during confirmed disruptions instead of waiting for hourly payouts.

**Community Disruption Reporting** — workers report accidents, restaurant issues, and platform glitches. Anti-abuse: a report only counts if at least 3 independent workers in the same zone report within 15 minutes. Repeated false reporters lose their reporting trust score, which also reduces their genuineness score.

**Voice Alerts** — all critical events spoken aloud so workers do not need to look at the screen while driving.

| Alert Type | Example |
|---|---|
| Risk | "Heavy traffic ahead. Expect delay." |
| Opportunity | "High demand nearby. Move for better earnings." |
| Guidance | "Waiting too long. Consider switching location." |
| System | "Disruption detected. Compensation initiated." |

- Activates at GPS speed > 5 km/h (covers slow-moving traffic)
- Workers fully stopped get a screen notification instead
- Max 1 alert per 2 minutes
- Multi-language: English, Hindi, Odia

---

## Fraud Detection

| Type | Signals Checked |
|---|---|
| Traffic | GPS path + frozen route snapshot + route deviation score |
| Weather | Cached API data + zone alerts + area activity patterns |
| Algorithm | Two-layer demand signal (cross-platform + restaurant activity) |
| Restaurant | Timestamps + tolerance buffer + multi-worker corroboration |
| All | Worker Performance Matrix + claim frequency vs activity pattern |

API failure policy: Claims are queued, never rejected due to API outage. Processed automatically once data is restored. Worker is notified of pending status throughout.

---

## Go-To-Market

Pilot city: Bhubaneswar — manageable scale, strong two-wheeler delivery density, lower acquisition cost than metros.

First cohort: Swiggy and Zomato delivery partners in Bhubaneswar.

First 1,000 workers: Partner with existing delivery partner WhatsApp communities in the city. First month free to build trust and collect real baseline data. Referral-based onboarding where existing workers invite peers.

Underwriting: InsureO is the technology and distribution layer. A licensed IRDAI-registered insurer underwrites the risk pool at commercial scale. Pilot operates as a fintech payout service under a tech service agreement, with a clear path to IRDAI registration as an insurance aggregator before full launch.

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
|   Route Snapshot Store, Fraud Detection  |
|   Claim Queue, API Fallback Handler      |
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
|   PostgreSQL (worker data, snapshots)    |
|   MongoDB (logs, alerts, claim queue)    |
+------------------------------------------+
                    |
        +-----------+------------+
        |                        |
+----------------+   +--------------------------+
| Razorpay       |   |  Mobile App              |
| Instant payout |   |  React, Dashboard, Map   |
| UPI AutoPay    |   |  TTS Engine, Voice       |
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
| Payments | Razorpay + UPI AutoPay |

---

## End-to-End Workflow

```
1.  Worker logs in -> consent checked -> baseline loaded (cold-start or personal)
2.  Real-time monitoring starts (GPS, orders, traffic, weather)
3.  At delivery dispatch -> route snapshot frozen and stored
4.  Disruption signal detected
5.  Efficiency score calculated — IF < 0.5, proceed
6.  External cause validated via API (fallback cache if API down)
7.  Route deviation checked against frozen snapshot (traffic claims)
8.  Two-layer demand signal checked (algorithm claims)
9.  Fraud + genuineness score + claim frequency checked
10. Valid -> claim auto-triggered
11. Payout = coverage % x loss, adjusted by reliability score
12. Razorpay processes instant payment
13. Voice alert: "Compensation initiated."
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
| Phase 1 | Core setup, baseline tracking, efficiency model, weather + traffic APIs, route snapshot store | In Progress |
| Phase 2 | Algorithm disruption (two-layer), restaurant delay, fraud layer, databases, consent framework | Planned |
| Phase 3 | Razorpay + UPI AutoPay, auto-claim pipeline, reliability scoring, WebSockets, claim queue | Planned |
| Phase 4 | Heatmap, zone switching, predictive alerts, micro-compensation, community reporting | Planned |
| Phase 5 | Voice alerts, multi-language, mobile UX polish, privacy dashboard for workers | Planned |
| Phase 6 | End-to-end testing, fraud stress tests, Isolation Forest retraining pipeline, beta launch | Planned |

---

## Team

Built at [Hackathon Name]

---

## License

This project was built for hackathon purposes. All rights reserved by the team.
