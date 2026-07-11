# Deutschlandticket Adoption Potential Assessment
Johnson & Johnson Medical GmbH — Norderstedt, Hamburg

##1. Objective

J&J wants to understand how likely employees are to adopt the Deutschlandticket (Germany's subsidized nationwide public transport pass) for their commute to the Norderstedt office. The analysis answers four practical questions:

- How long does each employee's commute currently take — by car vs. by public transport?
- How well-connected is each residential area to public transport?
- Which employees are the most realistic candidates to switch from car to public transport?
- What factors most strongly separate "likely to switch" employees from "unlikely to switch" employees?

##2. Data Sources

- HVV GTFS feed (stops.txt, stop_times.txt, trips.txt, calendar.txt, routes.txt, shapes.txt, transfers.txt)
- Official Hamburg public transport network — stop locations, schedules, routes
- Synthetic employee dataset (generated in-notebook)2,000 simulated employees, since real HR data wasn't available for this assessment. Home locations are distributed across realistic Hamburg-area districts (weighted toward areas with more commuters into Norderstedt)
- Google Maps Platform — Routes API with transit and driving travel times/distances

##3. Pipeline — Step by Step

Step 1 — Synthetic Employee Generation

Since no real employee address data was available, a synthetic population of 2,000 employees was generated with:

- Home location: randomly placed within 14 real Hamburg-area districts, weighted by realistic commuter density toward Norderstedt (e.g. Norderstedt itself and Langenhorn carry more weight than Pinneberg).
- Age group, gender, income group: assigned with realistic probability distributions (e.g. income skews toward "mid," age skews toward working-age bands).
- Current commute mode (car / public_transport / bike / walk): assigned conditional on income group where higher-income employees are modeled as more car-dependent and lower-income employees more transit-dependent.

This is synthetic data built to demonstrate the methodology — in a production setting, this step would be replaced with real (anonymized) employee address data.

Step 2 — Nearest Public Transport Stop

For each employee, the nearest GTFS stop to their home coordinates is found using haversine distance (great-circle distance), adjusted by a detour factor (1.35×) to approximate real walking distance vs. straight-line distance (people don't walk in straight lines — roads/paths add ~35% extra distance on average).

This determines each employee's realistic "first-mile" access point to the transit network.

Step 3 — Travel Time & Distance (Google Routes API)

For each employee, two real-world routes are computed to the workplace:

- Public transport route (transit mode, 08:00 AM weekday departure) — returns total travel time, distance, and number of transfers.
- Driving route (traffic-aware) — returns total travel time and distance by car.

Both are computed for the same reference date/time, so they're directly comparable ("how much longer does transit take vs. driving for this specific person").

Step 4 — Deutschlandticket Adoption Scoring

Each employee receives a willingness-to-adopt score, combining two types of signals:

- Feasibility (would transit realistically work for them?) — 60% of the score
- Time competitiveness: how close is transit travel time to driving time
- Transfer burden: fewer transfers = more attractive (penalty increases non-linearly per transfer)
- First-mile access: how far is the walk to their nearest stop
- Propensity (are they a realistic candidate to switch?) — 40% of the score

Current commute mode: car users represent the real opportunity (highest propensity); existing public transport users are already "converted" and excluded as prospects
Income group: lower-income employees are modeled as more responsive to a subsidized ticket's cost savings

These five factors are each normalized to a 0–1 scale and combined into a single weighted score, then bucketed into Low / Medium / High adoption potential.

Step 5 — Summary

Step 6 — Interactive Map

A Folium-based interactive HTML map visualizes:

- Workplace location
- HVV station locations
- Employees grouped into toggleable clusters by commute-time band (≤30 / 30-45 / 45-60 / 60+ min)
- High-adoption-potential employees highlighted as a distinct layer

Each layer can be toggled on/off independently, allowing focused exploration (e.g. "show me only the High-potential employees" or "show me only the 60+ minute commuters").

##4. Methodology Notes — Why Google Routes API Instead of Pure GTFS Routing

An initial approach attempted to compute exact multi-transfer journeys directly from the GTFS schedule data. This gives the most precise answer for an exact date, but was ultimately replaced with the Google Routes API for the final pipeline because:

- It handles real-world walking/transfer buffers and alternative routings automatically
- It provides both transit and driving estimates from the same consistent engine, enabling direct comparison
- It's more practical to scale and maintain than a custom multi-leg journey planner

The trade-off: results reflect a near-term representative date rather than the exact originally-requested date, and rely on Google's live routing rather than the raw HVV schedule.
