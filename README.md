# Microservice Dependency & Temporal Graph Analysis

## 1. Overview & Dataset

This study analyzes minute-level RPC telemetry from a 10-service e-commerce application over a **10-day period** (`2026-04-27` to `2026-05-07`, spanning **240 hours** / 14,146 unique timestamps).

**Objectives:**

1. Compare **static vs. temporal (hourly)** dependency topologies.
2. Identify **Magnet Services** (critical sinks with the highest call volumes).
3. Identify **Event Horizon (EH) Candidates** (intermediary nodes on shortest paths to magnets).
4. Compute **Magnet Exposure Ratios** to classify services into *High, Medium, and Low* risk zones.

**Key Columns:** `timestamp_bucket_utc`, `caller_service` ($u$), `callee_service` ($v$), `call_count` ($w_{uv}$), `error_count`, and latency metrics (`mean`, `p95`, `max`).

## 2. Static vs. Weighted Graph Analysis

Aggregating all calls over the 10-day window yields a static directed graph $G = (V, E)$ with **10 nodes**, **82 edges**, **0 self-loops**, and **41 reciprocal edge pairs**.

* **Static Unweighted Limitation:** Because edges are aggregated over 10 days, the static graph forms a single Strongly Connected Component (SCC) of size 10. Almost all nodes have an In/Out-Degree of 8–9 (except `adservice` at 5) and 9 transitive dependents, masking true runtime hierarchy.
* **Weighted Call Volume ($G_{\text{weighted}}$):** Weighting edges by total `call_count` reveals the true traffic sinks.

| Service | Static In/Out Degree | Static PageRank | Total Incoming Calls (10 Days) | Hourly Avg Rank | 
| ----- | ----- | ----- | ----- | ----- | 
| **`productcatalogservice`** | 9 / 9 | 0.1088 | **7,247,745** | **1.00** (Top 3 in 240/240 hrs) | 
| **`currencyservice`** | 9 / 9 | 0.1088 | **6,010,119** | **2.00** (Top 3 in 240/240 hrs) | 
| **`cartservice`** | 9 / 9 | 0.1088 | **1,471,658** | **3.00** (Top 3 in 240/240 hrs) | 
| `recommendationservice` | 9 / 9 | 0.1088 | 863,756 | 4.00 | 
| `adservice` | 5 / 5 | 0.0664 | 664,846 | 5.00 | 
| `shippingservice` | 8 / 8 | 0.0974 | 575,559 | 6.00 | 
| `checkoutservice` | 8 / 8 | 0.0974 | 57,738 | 7.24 | 
| `emailservice` | 8 / 8 | 0.0974 | 57,567 | 7.32 | 
| `paymentservice` | 8 / 8 | 0.0974 | 57,564 | 7.34 | 
| `frontend` | 9 / 9 | 0.1088 | 4,963 | 10.00 | 

> **Candidate Magnets ($M$):** `productcatalogservice`, `currencyservice`, and `cartservice` consistently rank #1, #2, and #3 in every single hour and are designated as the system's **Candidate Magnets**.

## 3. Temporal (Hourly) Topology & Dependency Exposure

Partitioning the data into **240 hourly graphs ($G_t$)** shows that runtime dependencies are sparse and dynamic compared to the static graph:

* **Active Edges per Hour:** Mean = $29.77 \pm 8.37$ (Min = $13$, Median = $29$, Max = $53$).
* **SCCs per Hour:** Mean = $3.97$ components (Largest SCC Mean = $7.03$ nodes).

### Hourly Dependency Exposure (Blast Radius)

Comparing **Direct Dependents** (predecessors) vs. **Transitive Dependents** (ancestors) across the 240 hourly graphs:

| Service | Mean Direct Dependents ($\pm \text{Std}$) | Mean Transitive Dependents ($\pm \text{Std}$) | Hourly Mean Calls ($\pm \text{Std}$) | 
| ----- | ----- | ----- | ----- | 
| `productcatalogservice` | $5.02 \pm 1.17$ | $6.35 \pm 1.30$ | $30,198.9 \pm 11,221.2$ | 
| `cartservice` | $4.22 \pm 1.25$ | $6.29 \pm 1.46$ | $6,131.9 \pm 2,255.0$ | 
| `frontend` | $3.81 \pm 1.76$ | $6.03 \pm 1.78$ | $20.7 \pm 16.0$ | 
| `shippingservice` | $3.51 \pm 1.19$ | $6.38 \pm 1.43$ | $2,398.2 \pm 890.8$ | 
| `currencyservice` | $3.51 \pm 1.38$ | $6.16 \pm 1.60$ | $25,042.2 \pm 9,222.4$ | 
| `recommendationservice` | $3.07 \pm 1.29$ | $6.13 \pm 1.60$ | $3,599.0 \pm 1,329.6$ | 
| `adservice` | $2.25 \pm 1.02$ | $6.43 \pm 1.53$ | $2,770.2 \pm 1,008.3$ | 
| `checkoutservice` | $1.55 \pm 0.85$ | $6.08 \pm 1.66$ | $240.6 \pm 92.1$ | 
| `paymentservice` | $1.43 \pm 0.69$ | $6.93 \pm 1.55$ | $239.9 \pm 91.6$ | 
| `emailservice` | $1.39 \pm 0.67$ | $6.94 \pm 1.55$ | $239.9 \pm 91.5$ | 

* **Insight:** Even leaf services with low direct in-degree (`paymentservice`, `emailservice`) have a high transitive blast radius ($\sim 6.9$ services) due to multi-hop call chains.

## 4. Event Horizon (EH) & Magnet Exposure Analysis

### 4.1 Shortest-Path Intermediary Participation

An **Event Horizon (EH) Candidate** is a non-magnet service $k$ that lies on the shortest path $P(s, m) = (s, \dots, k, \dots, m)$ between a non-magnet source $s \in N$ and a magnet target $m \in M$.

* In the **static graph**, all $s \to m$ paths are direct 1-hop edges (`intermediary appearances = 0`).
* In the **240 hourly graphs**, missing direct edges force routing through intermediaries (**1,997 total appearances**).

### 4.2 Magnet Exposure Ratio & Risk Zoning

For each non-magnet $s$, the hourly **Magnet Exposure Ratio** is the fraction of magnets reachable from $s$:

$$
E_t(s) = \frac{\vert{}\text{Descendants}_{G_t}(s) \cap M\vert{}}{\vert{}M\vert{}}, \quad (\vert{}M\vert{} = 3)
$$

Combining intermediary path participation and exposure ratios yields the final service classification:

| Non-Magnet Service ($s$) | Intermediary Path Pairs (Active Hrs) | Mean Exposure $\mu_E$ ($\pm \sigma_E$) | Exposed Hours | Zone `(0.4, 0.6)` | Zone `(0.6, 0.8)` | 
| ----- | ----- | ----- | ----- | ----- | ----- | 
| **`frontend`** | **575** (179 hrs) | **1.000** ($\pm 0.000$) | 240 / 240 | **High** | **High** | 
| **`checkoutservice`** | **95** (41 hrs) | **0.986** ($\pm 0.067$) | 240 / 240 | **High** | **High** | 
| **`recommendationservice`** | **154** (68 hrs) | **0.944** ($\pm 0.185$) | 240 / 240 | **High** | **High** | 
| **`shippingservice`** | **122** (79 hrs) | **0.717** ($\pm 0.449$) | 173 / 240 | **High** | **Medium** | 
| **`adservice`** | **10** (8 hrs) | **0.628** ($\pm 0.482$) | 152 / 240 | **High** | **Medium** | 
| **`paymentservice`** | **15** (8 hrs) | **0.144** ($\pm 0.350$) | 36 / 240 | **Low** | **Low** | 
| **`emailservice`** | **10** (6 hrs) | **0.142** ($\pm 0.349$) | 34 / 240 | **Low** | **Low** | 

*(Note: Cross-magnet intermediary appearances were also observed for `productcatalogservice` \[553 pairs\], `cartservice` \[216\], and `currencyservice` \[197\]).*

## 5. Key Takeaways

1. **Temporal Slicing is Essential:** Static graphs hide intermediary bottlenecks; hourly graphs reveal how traffic dynamically routes through bridge nodes.
2. **Core Magnets:** `productcatalogservice`, `currencyservice`, and `cartservice` absorb the highest call volumes in 100% of observed hours.
3. **High-Exposure Event Horizon:** `frontend` (primary bridge with 575 path pairs), `checkoutservice`, and `recommendationservice` have $\mu_E > 0.94$, meaning any magnet degradation immediately impacts them.
4. **Threshold Sensitivity:** Under stricter `(0.6, 0.8)` thresholds, `shippingservice` and `adservice` shift from *High* to *Medium Exposure*, accurately reflecting their intermittent coupling, while `paymentservice` and `emailservice` remain isolated in the *Low Exposure* zone.
