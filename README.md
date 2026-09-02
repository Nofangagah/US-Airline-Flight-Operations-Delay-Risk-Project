#  US Airline Flight Operations & Delay Risk Intelligence Dashboard

An end-to-end operational intelligence dashboard built with **Power BI**, backed by a data warehouse pipeline (**PostgreSQL / Staging & Analytics Marts**), analyzing over **3.68 million flights** to identify on-time performance drivers, route bottlenecks, and temporal delay escalation patterns.

---

##  Executive Summary

- **Total Flights Analyzed:** 3.68M scheduled operations
- **Industry On-Time Rate (OTP):** 81.39%
- **Industry Delay Rate:** 17.24%
- **Cancellation Rate:** 1.13%
- **Primary Delay Driver:** *Late Aircraft Delay* & *Carrier Delay* (>70% of total delay minutes across major carriers)

---

##  Architecture & Data Modeling

### 1. Data Pipeline & Schema

The data architecture follows a structured ETL staging and dimensional modeling workflow.

#### Raw / Staging

- `staging.flights`
  - Flight-level granular operational logs
  - Departure and arrival timestamps
  - Flight status
  - Delay breakdown categories
  - Cancellation indicators

#### Analytics Data Marts

- `analytics.airline_monthly_summary`
  - Monthly performance metrics
  - Aggregated OTP
  - Cancellation rates
  - Delay cause distributions

- `analytics.route_performance`
  - Origin-destination route volume
  - Distance
  - Delay rates
  - Delay severity

- `analytics.time_pattern_analysis`
  - Multi-dimensional temporal analysis
  - Aggregated by:
    - `airline_name`
    - `origin_airport`
    - `day_of_week`
    - `departure_hour`

### 2. Core SQL Aggregation Example

The following view supports the temporal delay-risk analysis used in the Power BI dashboard.

```sql
CREATE OR REPLACE VIEW analytics.v_time_pattern_analysis AS
SELECT 
    f.airline_name,
    f.origin_airport,
    f.day_of_week,
    CASE f.day_of_week
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
        WHEN 7 THEN 'Sunday'
    END AS day_name,
    (f.scheduled_departure / 100)::INT AS departure_hour,
    COUNT(*) AS total_flights,
    COUNT(*) FILTER (
        WHERE f.flight_status = 'DELAYED'
    ) AS delayed_flights,
    ROUND(
        COUNT(*) FILTER (
            WHERE f.flight_status = 'DELAYED'
        ) * 100.0 / NULLIF(COUNT(*), 0),
        2
    ) AS delay_rate_pct,
    ROUND(
        AVG(f.departure_delay) FILTER (
            WHERE f.flight_status = 'DELAYED'
        ),
        2
    ) AS avg_departure_delay_mins,
    ROUND(
        AVG(f.arrival_delay) FILTER (
            WHERE f.flight_status = 'DELAYED'
        ),
        2
    ) AS avg_arrival_delay_mins
FROM staging.flights f
WHERE f.is_cancelled = FALSE 
  AND f.scheduled_departure IS NOT NULL
GROUP BY 
    f.airline_name, 
    f.origin_airport, 
    f.day_of_week, 
    (f.scheduled_departure / 100)::INT;
```
 Dashboard Structure & Visual Breakdown

The dashboard is structured into three analytical pages:

├── Page 1: Executive Overview
│   └── Macro Performance & Root Causes
│
├── Page 2: Route & Geography
│   └── Spatial Bottlenecks & Pairwise Matrix
│
└── Page 3: Time Patterns
    └── Cascading Delay Risk Heatmap & Hourly Curves
Page 1 — Executive Overview
KPI Cards

Top-level macro health indicators:

Total Flights
On-Time Rate
Delay Rate
Cancellation Rate

These KPIs provide an immediate overview of overall network operational performance.

Monthly Flight Volume vs Performance Trends

A dual-axis chart tracking:

Monthly flight volume
Seasonal volume fluctuations
Monthly On-Time Performance (OTP)
Monthly delay-rate fluctuations

The visualization helps identify whether changes in operational performance coincide with seasonal changes in flight volume.

Airline On-Time Performance (OTP) Ranking

A horizontal benchmark ranking airlines based on On-Time Performance.

Industry Leaders
Hawaiian Airlines: 89.20%
Delta: 86.45%
Industry Laggard
Frontier: 74.02%

The benchmark highlights the performance gap between leading and underperforming carriers.

Root Cause Delay Distribution by Airline

A 100% stacked bar chart dissecting total delay minutes by major delay category:

Carrier Delay
NAS Delay
Weather Delay
Late Aircraft Delay
Security Delay

This visualization identifies which delay mechanisms contribute most heavily to operational disruption across airlines.

Page 2 — Route & Geography
US Departure Volume & Delay Intensity Shape Map

A state-level choropleth map showing geographic differences in flight delay rates.

The map uses the following risk classification:

🟢 Teal: Low Delay Risk
🟡 Yellow: Moderate Delay Risk
🔴 Red: High Delay Risk

This allows operational stakeholders to quickly identify geographic areas experiencing elevated delay pressure.

Top 10 High-Delay Routes

Routes are filtered using a minimum threshold of 1,000 flights to prevent low-volume routes from distorting the analysis.

Example high-risk corridors include:

LGA → CMH: 32.7% delay rate
RDU → LGA: 32.6% delay rate

The analysis highlights critical airport-to-airport corridors where operational bottlenecks are concentrated.

Comprehensive Route Performance Matrix

A detailed route-level table containing:

Metric	Description
Origin Airport	Departure airport
Destination Airport	Arrival airport
Distance	Route distance
Flight Volume	Number of scheduled flights
Delay Rate	Percentage of delayed flights
Average Arrival Delay	Average arrival delay duration

The matrix enables granular drill-down from network-level performance into individual flight corridors.

Page 3 — Time Patterns & Delay Risk Heatmap
Delay Risk Heatmap

A matrix cross-referencing:

Departure Hour: 00:00–23:00
Day of Week: Monday–Sunday

Conditional formatting uses a three-color gradient to identify periods of elevated operational risk.

This visualization is designed to reveal recurring temporal patterns that may not be visible in aggregate performance metrics.

Hourly Delay Escalation Curve

An area/line chart demonstrating the daily delay escalation pattern.

The analysis shows how delay risk evolves throughout the operating day:

Morning troughs → Afternoon escalation → Evening peaks

The curve provides evidence of potential cascading delays caused by aircraft turnaround cycles and accumulated operational disruptions.

Flight Volume vs. Delay Rate by Day of Week

A mixed column and line visualization comparing:

Daily flight volume
Daily delay rate

This allows stakeholders to assess whether operational risk increases alongside passenger and flight demand.

🔍 Key Findings & Business Insights
1. The Compounding Cascading Delay Effect

Flights departing between 06:00–08:00 carry the lowest observed delay risk:

6.9%–10.5% delay rate

As the operating day progresses, unresolved schedule buffers and aircraft turnaround delays can accumulate across successive rotations.

Delay risk reaches its highest level between 17:00–21:00:

25.2%–28.7% delay rate
Approximately 4× higher risk compared with early-morning operations

This pattern suggests that operational delays can propagate through the network as aircraft and crews move through multiple daily rotations.

2. Weekly Operational Stress Points

Thursday and Friday experience the highest operational friction.

Day	Delay Rate
Thursday	18.63%
Friday	18.26%
Saturday	16.05%

The elevated Thursday and Friday delay rates are associated with periods of increased operational pressure, including:

Business travel demand
Weekend departure demand
Higher traffic volume
Increased aircraft utilization

Saturday exhibits the highest operational stability, with a 16.05% delay rate, alongside comparatively reduced commercial volume.

3. Hub Congestion Concentration

Operational risk is concentrated around major aviation hubs.

The analysis particularly identifies:

New York — LGA
Chicago — ORD

as important nodes associated with a significant proportion of high-risk routes exceeding 30% delay likelihood.

This indicates that network-level delay risk is not evenly distributed and can be heavily influenced by congestion at major hub airports.

 Strategic Operational Recommendations
Strategy	Operational Action	Expected Business Impact
Turnaround Buffer Adjustment	Extend scheduled turnaround buffers by 15–20 minutes for flights departing after 16:00 at major hub airports such as LGA, ORD, and LAX.	Mitigates cascading delays caused by Late Aircraft Delay during late-evening rotations.
Fleet Rotation Isolation	Decouple aircraft rotations serving congested Northeast corridors from independent regional sub-networks.	Reduces the probability of localized airspace or weather disruptions propagating across national line operations.
Peak-Day Dynamic Staffing	Allocate reserve aircraft and on-call ground crew standby windows during Thursday/Friday afternoon departure banks.	Absorbs peak operational demand shocks and helps protect target OTP metrics.
Off-Peak Demand Incentivization	Introduce tiered fare pricing strategies favoring early-morning departure banks between 06:00–08:00.	Helps balance runway demand and reduce peak-hour ramp congestion.
🛠️ Tech Stack & Prerequisites
Data Storage & ETL
PostgreSQL 15+
DBeaver
BI & Visual Analytics
Microsoft Power BI Desktop
Data Architecture
Raw / Staging Layer
Analytics Data Marts
SQL Views
Power BI Semantic Model
Executive Dashboard
Color Palette
Clean Executive Modern
Color	Hex	Usage
Teal	#0F766E	Primary / Low Risk
Light Teal	#CCFBF1	Background / Supporting Elements
Soft Yellow	#FDE047	Moderate Risk
Rose Red	#E11D48	High Risk / Alerts
Slate	#1E293B	Text / Headers
📌 Project Objective

The dashboard transforms 3.68M+ flight-level operational records into actionable operational intelligence across three analytical dimensions:

Performance → Geography → Time

By combining:

Airline performance benchmarking
Route-level bottleneck analysis
Geographic delay concentration
Temporal delay-risk modeling
Root-cause delay analysis

the dashboard enables operational stakeholders to identify:

Where delays are concentrated, when operational risk escalates, and why delays propagate across the airline network.

Business Value

The dashboard is designed to support operational decision-making in four key areas:

1. Performance Monitoring

Identify airlines, periods, and operating conditions with below-target On-Time Performance.

2. Network Optimization

Detect high-risk routes and airports that contribute disproportionately to overall delay exposure.

3. Capacity & Resource Planning

Identify high-risk departure windows and days that require additional operational capacity.

4. Delay Mitigation

Use delay root-cause and temporal patterns to support targeted interventions around aircraft turnaround, staffing, fleet rotations, and hub congestion.

 Analytical Framework
 ```
                    ┌───────────────────────┐
                    │   3.68M+ Flights      │
                    │   Operational Data    │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │ PostgreSQL Staging    │
                    │ staging.flights       │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │ Analytics Data Marts  │
                    ├───────────────────────┤
                    │ Airline Performance   │
                    │ Route Performance     │
                    │ Time Pattern Analysis │
                    └───────────┬───────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │     Power BI Model    │
                    └───────────┬───────────┘
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
     ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
     │ Performance  │   │ Geography    │   │ Time Risk    │
     │ Analysis     │   │ Analysis     │   │ Analysis     │
     └──────────────┘   └──────────────┘   └──────────────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                ▼
                    ┌───────────────────────┐
                    │ Operational           │
                    │ Intelligence          │
                    └───────────────────────┘
```
Conclusion

The US Airline Flight Operations & Delay Risk Intelligence Dashboard provides a comprehensive view of airline operational performance by combining 3.68M+ flight records, PostgreSQL-based analytical data marts, and Power BI visualization.

The analysis demonstrates that delay risk is strongly influenced by time of departure, aircraft rotation cycles, hub congestion, route characteristics, and weekly demand patterns.

The resulting intelligence can support more targeted operational decisions, particularly around:

Turnaround scheduling
Hub capacity management
Fleet rotation planning
Peak-period staffing
Route risk monitoring
Demand distribution

Ultimately, the dashboard shifts flight-delay analysis from simple historical reporting toward proactive operational risk intelligence.