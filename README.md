# Yatrafy — Trip Planner: Technical Report

> **Project**: AttractionClustering / Yatrafy Prototype  
> **Stack**: Vanilla JavaScript, Vite, Google Maps JavaScript API, Google Places API (New)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Data Acquisition Layer](#3-data-acquisition-layer)
4. [Attraction Enrichment & Classification](#4-attraction-enrichment--classification)
5. [Trip Planning Algorithm](#5-trip-planning-algorithm)
6. [Hotel & Dining Subsystem](#6-hotel--dining-subsystem)
7. [Caching Strategy](#7-caching-strategy)
8. [UI & Interaction Model](#8-ui--interaction-model)
9. [Mathematical Formulations Summary](#9-mathematical-formulations-summary)

---

## 1. Project Overview

Yatrafy is a client-side trip planning application that generates optimized multi-day itineraries for city tourism. Given a set of attractions fetched from Google Places API, the system:

1. **Classifies** each attraction into environment types (Indoor / Outdoor / Mixed), categories (Cultural / Nature / Food), and computes continuous scores for heat exposure, walking intensity, popularity, and overall visit priority.
2. **Plans** an optimal day-by-day itinerary using a greedy nearest-neighbor algorithm with multi-objective scoring, subject to time, distance, heat, and walking constraints.
3. **Integrates** hotel selection and restaurant discovery with budget-aware, location-biased fetching and spatial caching.
4. **Renders** the itinerary on a Google Maps instance with interactive markers, day tabs, per-stop timelines, and inline manipulation (swap, add, remove, reorder).

All computation runs entirely in the browser — no backend server is needed beyond the Vite dev proxy for CORS-free access to Google APIs.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser (Client)                     │
├─────────────┬─────────────┬────────────┬────────────────┤
│  main.js    │  planner.js │  UI Layer  │   CSS Styles   │
│ (Landing)   │ (Planner)   │ renderCards│ style / planner │
├─────────────┴─────────────┴────────────┴────────────────┤
│                    Utility Layer                         │
│  enrichAttractions.js │ tripPlanner.js │ hotelDining.js  │
│       cache.js        │                                  │
├──────────────────────────────────────────────────────────┤
│                    API Layer                              │
│              googlePlaces.js                              │
│  (Google Places API New — Text Search, Nearby Search)     │
├──────────────────────────────────────────────────────────┤
│                  Vite Dev Proxy                           │
│    /api/places/* → places.googleapis.com (CORS bypass)    │
└──────────────────────────────────────────────────────────┘
```

| Module | Responsibility |
|--------|---------------|
| `googlePlaces.js` | HTTP calls to Google Places API (New), field masking, response normalization |
| `enrichAttractions.js` | NLP keyword classification, scoring (heat, walking, visit, category) |
| `tripPlanner.js` | Haversine, travel time, greedy planner, timeline generation |
| `hotelDining.js` | Hotel management, restaurant discovery, budget-tier fetching |
| `cache.js` | localStorage TTL-based caching with LRU eviction |
| `main.js` | Landing page: search, category pill filters, include toggles |
| `planner.js` | Planner page: constraints, map rendering, sidebar, popups |

---

## 3. Data Acquisition Layer

### 3.1 Google Places API (New) — Text Search

All place data is fetched via the **Google Places API (New)** using the `searchText` endpoint.

**Request structure:**

```json
{
  "textQuery": "tourist attractions in Chennai",
  "languageCode": "en",
  "maxResultCount": 20
}
```

**Field mask** (sent as `X-Goog-FieldMask` header):
```
places.displayName, places.formattedAddress, places.rating,
places.userRatingCount, places.photos, places.types,
places.editorialSummary, places.googleMapsUri,
places.location, places.regularOpeningHours, places.priceLevel
```

The field mask is critical for API cost optimization — Google charges based on fields requested, and this mask requests only Basic + Contact fields.

### 3.2 Response Normalization

Each raw Google Place response is normalized into a uniform `Attraction` object:

```javascript
{
    id:           "google_<slugified_name>",
    name:         place.displayName.text,
    address:      place.formattedAddress,
    rating:       place.rating,          // 1.0 – 5.0
    ratingCount:  place.userRatingCount,  // integer
    photo:        "https://places.googleapis.com/v1/{photoRef}/media?maxWidthPx=600&key={KEY}",
    types:        ["hindu temple", "tourist attraction"],
    lat:          place.location.latitude,
    lng:          place.location.longitude,
    todayHours:   "9:00 AM – 5:00 PM",
    priceLevel:   "PRICE_LEVEL_MODERATE",
    mapUrl:       place.googleMapsUri,
}
```

### 3.3 Location-Biased Restaurant Search

For restaurant discovery near itinerary attractions, a **circular location bias** is applied:

```json
{
  "textQuery": "restaurants and food",
  "locationBias": {
    "circle": {
      "center": { "latitude": 13.0827, "longitude": 80.2707 },
      "radius": 2000
    }
  }
}
```

The **radius of 2 km** balances result relevance with coverage. This creates a weighted search region — Google prioritizes results within the circle but may return results slightly outside.

### 3.4 Budget-Tier Queries

Budget filtering is implemented via **semantic query differentiation** rather than post-filtering:

| Tier | Hotel Query | Restaurant Query |
|------|-------------|-----------------|
| Any | `hotels and resorts in {city}` | `restaurants and food` |
| Budget ($) | `budget hotels and hostels in {city}` | `cheap restaurants and street food` |
| Mid ($$) | `mid-range hotels in {city}` | `mid-range restaurants and dining` |
| Premium ($$$) | `luxury hotels and 5 star resorts in {city}` | `fine dining and luxury restaurants` |

This approach leverages Google's NLP understanding of price semantics to return tier-appropriate results, which is more effective than filtering on `priceLevel` alone (which many places lack).

---

## 4. Attraction Enrichment & Classification

The enrichment pipeline (`enrichAttractions.js`) transforms raw place data into scored, classified attractions. This is a **keyword-based NLP classifier** — a lightweight, deterministic alternative to ML models.

### 4.1 Text Feature Extraction

For each attraction, a combined text feature is computed:

```
allText = types.join(' ').toLowerCase() + ' ' + name.toLowerCase()
```

For example, an attraction with `types: ["hindu temple", "place of worship"]` and `name: "Kapaleeshwarar Temple"` yields:

```
allText = "hindu temple place of worship kapaleeshwarar temple"
```

### 4.2 Environment Classification

A **rule-based classifier** with three non-exclusive keyword dictionaries:

| Environment | Keywords |
|------------|----------|
| **Indoor** | museum, gallery, art, theatre, auditorium, aquarium, planetarium, library, mall, shopping, cinema, spa, cafe, restaurant |
| **Outdoor** | beach, park, garden, lake, river, viewpoint, trail, hiking, waterfall, mountain, zoo, marina, pier, playground, stadium, square |
| **Mixed** | temple, church, mosque, cathedral, fort, palace, monument, memorial, heritage, ruins, amusement park, market, bazaar |

**Classification logic:**

```
if (hasMixed)                   → "Indoor + Outdoor"
else if (hasIndoor && hasOutdoor) → "Indoor + Outdoor"
else if (hasIndoor)              → "Indoor"
else if (hasOutdoor)             → "Outdoor"
else                             → "Indoor + Outdoor"  (default for unknowns)
```

The "Mixed" dictionary takes precedence because places like temples have both indoor prayer halls and outdoor courtyards — a single environment label would be misleading.

### 4.3 Category Classification

Three binary categories are assigned using keyword matching:

$$
\text{category}(a) = \begin{bmatrix} \text{cultural} \\ \text{nature} \\ \text{food} \end{bmatrix} = \begin{bmatrix} \mathbb{1}[\text{allText} \cap \text{CULTURAL\_KW} \neq \emptyset] \\ \mathbb{1}[\text{allText} \cap \text{NATURE\_KW} \neq \emptyset] \\ \mathbb{1}[\text{allText} \cap \text{FOOD\_KW} \neq \emptyset] \end{bmatrix}
$$

Where $\mathbb{1}[\cdot]$ is the indicator function. If all three are 0 (no keywords match), the attraction defaults to `cultural = 1`.

**Category keyword groups:**

| Category | Sample Keywords |
|----------|----------------|
| **Cultural** | temple, museum, gallery, fort, palace, monument, heritage, historic, ruins, theatre |
| **Nature** | beach, park, garden, lake, trail, waterfall, mountain, zoo, marina, aquarium |
| **Food** | cafe, restaurant, bar, food, bakery, market, mall, shopping, dining |

An attraction can belong to **multiple categories** simultaneously (e.g., a beachside cafe would be `nature=1, food=1`).

### 4.4 Heat Exposure Index

Each attraction is assigned a **heat exposure score** $h \in [0, 1]$ based on keyword matching against a lookup table:

$$
h(a) = \max_{kw \in \text{allText}} \text{HEAT\_SCORES}[kw]
$$

If no keyword matches, the default is $h = 0.5$.

**Heat score lookup (selected values):**

| Keyword | Score | Keyword | Score | Keyword | Score |
|---------|-------|---------|-------|---------|-------|
| beach | 1.00 | hiking | 0.90 | park | 0.85 |
| zoo | 0.80 | viewpoint | 0.75 | plaza | 0.70 |
| waterfall | 0.65 | monument | 0.60 | temple | 0.50 |
| market | 0.45 | church | 0.30 | museum | 0.10 |
| theatre | 0.05 | restaurant | 0.05 | spa | 0.05 |

**Severity classification:**

| Range | Label | Color | Emoji |
|-------|-------|-------|-------|
| $h \geq 0.8$ | Very High | `#ef4444` | 🔥 |
| $0.6 \leq h < 0.8$ | High | `#f97316` | 🌡️ |
| $0.4 \leq h < 0.6$ | Moderate | `#eab308` | ⛅ |
| $0.2 \leq h < 0.4$ | Low | `#22c55e` | 🌤️ |
| $h < 0.2$ | Minimal | `#3b82f6` | ❄️ |

### 4.5 Walking Intensity Score

Each attraction's walking requirement is scored using $w \in [0, 1]$:

$$
w(a) = \max_{kw \in \text{allText}} \text{WALKING\_SCORES}[kw]
$$

Default (no match): $w = 0.4$.

**Walking score lookup (selected values):**

| Keyword | Score | Keyword | Score | Keyword | Score |
|---------|-------|---------|-------|---------|-------|
| hiking | 1.00 | trail | 1.00 | safari | 0.95 |
| zoo | 0.90 | beach | 0.85 | park | 0.80 |
| ruins | 0.80 | garden | 0.75 | waterfall | 0.70 |
| fort | 0.65 | palace | 0.60 | museum | 0.50 |
| mall | 0.45 | temple | 0.35 | gallery | 0.35 |
| theatre | 0.20 | cafe | 0.15 | restaurant | 0.10 |

**Walking difficulty labels:**

| Range | Label | Icon |
|-------|-------|------|
| $w \geq 0.8$ | Heavy | 🥾 |
| $0.5 \leq w < 0.8$ | Moderate | 🚶 |
| $w < 0.5$ | Light | 🪑 |

### 4.6 Visit Duration Estimation

Duration is estimated in two stages:

**Stage 1 — Base duration** from keyword matching:

$$
d_{\text{base}}(a) = \text{DURATION\_ESTIMATES}[\text{first matching keyword}]
$$

| Keyword | Minutes | Keyword | Minutes | Keyword | Minutes |
|---------|---------|---------|---------|---------|---------|
| beach | 90 | museum | 120 | amusement | 180 |
| zoo | 150 | hiking | 150 | temple | 60 |
| fort | 90 | gallery | 75 | park | 45 |
| viewpoint | 20 | memorial | 25 | church | 40 |
| safari | 180 | theatre | 120 | cafe | 45 |

Default: 60 minutes.

**Stage 2 — Popularity multiplier** based on `ratingCount`:

$$
d(a) = d_{\text{base}} \times \begin{cases} 1.30 & \text{if } \text{ratingCount} > 10{,}000 \\ 1.15 & \text{if } 5{,}000 < \text{ratingCount} \leq 10{,}000 \\ 0.80 & \text{if } \text{ratingCount} < 500 \\ 1.00 & \text{otherwise} \end{cases}
$$

**Rationale**: Popular places have more to see (larger exhibits, longer queues), while obscure places warrant shorter visits.

### 4.7 Visit Score (Priority Ranking)

The core ranking metric combines **popularity** and **quality**:

$$
\text{visitScore}(a) = 0.6 \times \tilde{p}(a) + 0.4 \times r(a)
$$

Where:

**Normalized popularity** uses log-scaling to compress the heavy-tailed rating count distribution:

$$
p(a) = \ln(\text{ratingCount}(a) + 1)
$$

$$
\tilde{p}(a) = \frac{p(a) - \min_i p(a_i)}{\max_i p(a_i) - \min_i p(a_i)}
$$

This yields $\tilde{p} \in [0, 1]$ via min-max normalization. The $\ln(x+1)$ transform is critical — without it, viral attractions with 50,000+ reviews would dominate, and niche gems with 200 reviews would never appear.

**Rating score:**

$$
r(a) = \frac{\text{rating}(a)}{5.0}
$$

The 60/40 weight split emphasizes popularity because:
- A 4.2★ attraction with 10,000 reviews is more reliably good than a 4.8★ with 15 reviews
- Tourist attractions with high review counts tend to be genuinely significant landmarks
- The log transform already prevents runaway dominance

---

## 5. Trip Planning Algorithm

### 5.1 Problem Formulation

The trip planning problem is a variant of the **Orienteering Problem (OP)** — a generalization of the Traveling Salesman Problem where not all nodes need to be visited, and each node has a reward (visit score). The objective is to maximize total reward subject to a budget constraint (time/distance).

Formally:

$$
\max \sum_{i \in S} \text{score}(a_i) \quad \text{subject to} \quad T(S) \leq T_{\max}, \; D(S) \leq D_{\max}
$$

Where $S$ is the selected subset of attractions, $T(S)$ is total time (travel + visit), and $D(S)$ is total distance.

The OP is NP-hard in general. We use a **greedy nearest-neighbor heuristic** which runs in $O(n^2)$ time per day (where $n$ is the number of candidate attractions).

### 5.2 Haversine Distance Formula

Inter-point distances are computed using the **Haversine formula**, which gives the great-circle distance between two points on a sphere:

$$
d = 2R \cdot \arctan2\left(\sqrt{a}, \sqrt{1-a}\right)
$$

Where:

$$
a = \sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_1) \cdot \cos(\phi_2) \cdot \sin^2\left(\frac{\Delta\lambda}{2}\right)
$$

- $\phi_1, \phi_2$ = latitudes in radians
- $\Delta\phi = \phi_2 - \phi_1$ = latitude difference
- $\Delta\lambda = \lambda_2 - \lambda_1$ = longitude difference
- $R = 6{,}371$ km (Earth's mean radius)

**Why Haversine over Euclidean?** At Chennai's latitude (~13°N), 1° longitude ≈ 109.6 km while 1° latitude ≈ 110.6 km. Euclidean distance in lat/lng coordinates would be inaccurate by ~3% due to the cosine projection effect. The Haversine is exact for spherical geometry.

### 5.3 Travel Time Estimation

Travel time between two points is estimated as:

$$
t_{\text{travel}} = \frac{d_{\text{haversine}}}{v_{\text{avg}}} \times 60 \quad \text{(minutes)}
$$

Where $v_{\text{avg}} = 25$ km/h — a conservative estimate for city driving in Chennai accounting for:
- Dense urban traffic (avg 15–20 km/h during peak)
- Moderate traffic (25–35 km/h off-peak)
- Auto-rickshaw/taxi navigation patterns
- Time to find parking/entrance

### 5.4 Greedy Nearest-Neighbor with Multi-Objective Scoring

For each day, the algorithm iteratively selects the best unvisited attraction:

```
while (candidates exist):
    for each unvisited attraction a:
        if HARD_FILTERS(a) fail → skip
        compute SCORE(a)
    select a* = argmax(SCORE)
    add a* to day plan
    update position, time, distance
```

#### 5.4.1 Hard Filters (Constraint Satisfaction)

An attraction is excluded if **any** of these constraints are violated:

| Constraint | Formula | Description |
|-----------|---------|-------------|
| Avoid list | $a.\text{id} \in \text{avoidIds}$ | User-specified exclusion |
| Heat | $h(a) > h_{\text{max}}$ | Exceeds day's heat tolerance |
| Walking | $w(a) > w_{\text{max}}$ | Exceeds day's walking tolerance |
| Time | $t_{\text{used}} + t_{\text{travel}} + d_{\text{visit}} > T_{\text{max}}$ | Would exceed daily time budget |
| Distance | $d_{\text{used}} + d_{\text{travel}} > D_{\text{max}}$ | Would exceed daily distance budget |
| End time (last day) | $t_{\text{start}} + t_{\text{used}} + t_{\text{travel}} + d_{\text{visit}} + t_{\text{return}} > t_{\text{end}}$ | Can't return to base by end time |

#### 5.4.2 Multi-Objective Scoring Function

For candidates that pass hard filters, the selection score is a **weighted linear combination**:

$$
\text{score}(a) = 0.35 \cdot V(a) + 0.30 \cdot C(a) + 0.20 \cdot P(a) + 0.15 \cdot W(a)
$$

**Component breakdown:**

**Visit Score** ($V$): Pre-computed attraction priority from §4.7.

$$
V(a) = \text{visitScore}(a) \in [0, 1]
$$

**Category Alignment** ($C$): Dot product of attraction's category vector with user preference weights, normalized:

$$
C(a) = \frac{\vec{c}(a) \cdot \vec{w}}{\max(w_{\text{cultural}}, w_{\text{nature}}, w_{\text{food}}, 0.01)}
$$

Where $\vec{c}(a) = [\text{cultural}, \text{nature}, \text{food}]$ (each 0 or 1) and $\vec{w} = [w_c, w_n, w_f]$ are user-configured slider values (0–1). The denominator normalizes by the maximum user weight to keep $C \in [0, 1]$.

**Proximity Bonus** ($P$): Inverse distance with a smoothing constant to avoid division by zero:

$$
P(a) = \frac{1}{d(a) + 0.5}
$$

Where $d(a)$ is the Haversine distance from the current position. This creates a strong preference for nearby attractions — an attraction 0.5 km away gets $P = 1.0$, while one 10 km away gets $P \approx 0.095$.

**Walking Comfort** ($W$): Inverted walking score (prefer less walking):

$$
W(a) = 1 - w(a)
$$

#### 5.4.3 Must-Visit Boost

Must-visit attractions receive a massive additive boost:

$$
\text{score}_{\text{final}}(a) = \text{score}(a) + \begin{cases} 1000 & \text{if } a.\text{id} \in \text{mustVisitIds} \\ 0 & \text{otherwise} \end{cases}
$$

This effectively guarantees must-visit places are selected first (since normal scores are in $[0, 1]$), while still allowing the greedy algorithm to pick the optimal insertion order among them.

#### 5.4.4 Weight Design Rationale

| Component | Weight | Justification |
|-----------|--------|--------------|
| Visit Score | 35% | Most important — ensures high-quality, popular places are prioritized |
| Category Alignment | 30% | Strong user personalization — if a user wants nature, temples shouldn't dominate |
| Proximity | 20% | Route efficiency — prevents excessive backtracking, keeps travel time low |
| Walking Comfort | 15% | Comfort factor — prevents a day full of heavy-walking attractions |

### 5.5 Day Plan Construction

Each day follows a **depot-return** structure (starting and ending at the hotel/airport):

```
Base → [Travel] → Attraction₁ → [Travel] → Attraction₂ → ... → [Travel] → Base
```

**Timeline generation** converts the sequence into a minute-by-minute schedule:

$$
t_{\text{arrive}}(a_k) = t_{\text{start}} + \sum_{i=1}^{k} \left(t_{\text{travel}}(a_{i-1} \to a_i) + d_{\text{visit}}(a_{i-1})\right)
$$

Where $a_0$ is the base point, and $t_{\text{start}}$ is 9:00 AM by default (configurable for Day 1 and Day 3's end time).

### 5.6 Per-Day Constraint Overrides

Each day can have **independent constraint overrides**:

```javascript
dayConstraints = [
    { dailyMinutes: 360, maxDistanceKm: 60, maxHeat: 0.7 },  // Day 1: short day
    null,  // Day 2: use global defaults
    { dailyMinutes: 480, maxDistanceKm: 100, maxHeat: 1.0 },  // Day 3: full day
]
```

This enables asymmetric planning — e.g., shorter first day after a late flight, or a relaxed last day before departure.

### 5.7 Algorithmic Complexity

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| Enrichment | $O(n \cdot k)$ | $n$ attractions, $k$ keywords |
| Greedy selection (per day) | $O(n^2)$ | Outer loop: max $n$ picks; inner loop: scan all $n$ candidates |
| Haversine (per call) | $O(1)$ | Constant-time trigonometry |
| Total planning | $O(D \cdot n^2)$ | $D$ days × $n^2$ selection |
| Timeline recalculation | $O(m)$ | $m$ attractions in a day |

For typical inputs ($n \approx 60$, $D = 3$), total planning completes in **< 5ms** on modern hardware.

---

## 6. Hotel & Dining Subsystem

### 6.1 Hotel Selection & Trip Base Point

The trip's start/end point is dynamic:
- **Default**: Chennai Airport (`lat: 12.9941, lng: 80.1709`)
- **When hotel selected**: Hotel's geographic coordinates

When the hotel changes, the entire itinerary is regenerated — all travel times, distances, and attraction ordering may change since the Haversine distances from the new base point differ.

### 6.2 Restaurant Discovery Algorithm

For each day's itinerary, restaurants are discovered using a **per-attraction proximity search**:

```
For each attraction a in Day D:
    gridKey = (lat.toFixed(2), lng.toFixed(2))   // spatial grid cell
    cacheKey = "restaurants_{gridKey}_{tier}"
    
    if cached → use cached results
    else → fetch from Google API with locationBias(a.lat, a.lng, 2000m)
           cache results under gridKey
    
    deduplicate by restaurant ID
```

**Grid-based caching key**: `lat.toFixed(2)` rounds to 2 decimal places (~1.1 km resolution). Two attractions within the same ~1 km grid cell share restaurant results, avoiding redundant API calls.

### 6.3 Restaurant Insertion into Itinerary

When a restaurant is added to a day's itinerary from the map or modal:

1. **Create attraction-like entry**: The restaurant is wrapped with default values:
   - `durationMinutes = 45` (meal duration)
   - `environment = "Indoor"`, `heatExposure = 0.1`
   - `walkingScore = 0` (minimal walking)
   - `visitScore = rating / 5`

2. **Find insertion point**: The algorithm finds the existing attraction closest to the restaurant:

$$
k^* = \arg\min_{k} \; d_{\text{haversine}}(a_k, r)
$$

3. **Insert after closest**: The restaurant is inserted at position $k^* + 1$ in the day's attraction list.

4. **Recalculate timeline**: All subsequent travel times and arrival times are recomputed.

### 6.4 Hotel Food Detection

Hotels are checked for food service capability:

```javascript
FOOD_TYPES = ['restaurant', 'meal_delivery', 'meal_takeaway', 'food', 'cafe']
hotel.servesFood = hotel.types.some(t => FOOD_TYPES.includes(t))
```

If the selected hotel serves food, it appears as the first option in all meal lists with an `isHotelDining` flag.

---

## 7. Caching Strategy

### 7.1 Architecture

The cache uses **localStorage** as a key-value store with time-based expiration:

```javascript
key    = "attraction_cache_{source}_{city}"
value  = { data: [...], cachedAt: timestamp, expiresAt: timestamp }
TTL    = 24 hours (default)
```

### 7.2 Cache Key Hierarchy

| Data Type | Cache Key Pattern | TTL |
|-----------|------------------|-----|
| Base attractions | `google_chennai` | 24h |
| Malls | `google_malls_chennai` | 24h |
| Cafes | `google_cafes_chennai` | 24h |
| Markets | `google_markets_chennai` | 24h |
| Hotels (by tier) | `google_hotels_{tier}_chennai` | 24h |
| Restaurants (by grid + tier) | `google_restaurants_{lat}_{lng}_{tier}_chennai` | 24h |

### 7.3 Eviction Strategy

On storage-full errors, the cache performs **eager expiration cleanup**:

```
for each key in localStorage that starts with "attraction_cache_":
    if current_time > entry.expiresAt:
        remove(key)
```

If cleanup frees enough space, the write is retried. Otherwise, the entry is silently dropped (graceful degradation).

### 7.4 Credit Conservation

The caching strategy is designed to minimize Google Places API credits:

1. **Per-tier caching**: Changing hotel budget from "$" to "$$" and back to "$" reuses the original cached results — no refetch.
2. **Grid-based restaurant deduplication**: Two nearby attractions (~1 km apart) share restaurant results.
3. **On-page caching**: Results cached on the landing page carry over to the planner page via shared localStorage.

---

## 8. UI & Interaction Model

### 8.1 Landing Page Flow

```
[City Input] → [Search] → [Google API Fetch] → [Enrich] → [Cache] → [Render Cards]
                                                     ↓
                                           Category Pill Filters
                                           Include Toggles (Malls/Cafes/Markets)
```

**Category filtering**: Uses the same keyword-matching approach as enrichment. Each pill activates a keyword group (Beach, Temple, Museum, Park, Fort, Zoo, Market, Theatre), and attractions are shown only if their `allText` contains a keyword from any active category.

**Deduplication**: When extra categories (malls, cafes, markets) are toggled on, results are merged with base attractions. Duplicates are removed using case-insensitive name matching:

$$
\text{key}(a) = \text{trim}(\text{lowercase}(a.\text{name}))
$$

### 8.2 Planner Page Layout

```
┌────────────────────────────────────────────────────────────────┐
│ Topbar: Start · Time · Distance · [Stay] [Meals] [⚙ Advanced]│
├──────────────────────────────────────────┬─────────────────────┤
│                                          │   3-Day Itinerary   │
│           Google Maps                    │   [Day 1|2|3] tabs  │
│    (Attraction + Restaurant markers)     │   Per-stop timeline  │
│                                          │   Time constraints   │
│                                          │   Swap/Add/Remove    │
│                                          │                     │
├──────────────────────────────────────────┴─────────────────────┤
│ Popups: Advanced Settings | Stay | Meals | Must-Visit/Avoid    │
└────────────────────────────────────────────────────────────────┘
```

### 8.3 Map Interaction

- **Attraction markers**: Colored by day (Day 1: `#6c63ff`, Day 2: `#e040fb`, Day 3: `#ff6b9d`)
- **Restaurant markers**: Yellow dots (`#f59e0b`), scale 8px, with info window showing rating + "Add to Day X" button
- **Route polylines**: Day-colored lines connecting stops in order
- **Click interaction**: Clicking an attraction highlights its nearby restaurants; clicking a day tab re-renders the relevant markers

---

## 9. Mathematical Formulations Summary

### 9.1 Distance & Time

$$
d_{\text{haversine}}(\phi_1, \lambda_1, \phi_2, \lambda_2) = 2R \cdot \arctan2\left(\sqrt{a}, \sqrt{1-a}\right)
$$

$$
t_{\text{travel}} = \left\lfloor \frac{d}{25} \times 60 \right\rceil \; \text{minutes}
$$

### 9.2 Scoring Functions

$$
\text{visitScore} = 0.6 \cdot \frac{\ln(n_r + 1) - \min}{\max - \min} + 0.4 \cdot \frac{r}{5}
$$

$$
\text{selectionScore} = 0.35 \cdot V + 0.30 \cdot \frac{\vec{c} \cdot \vec{w}}{\max(\vec{w})} + 0.20 \cdot \frac{1}{d + 0.5} + 0.15 \cdot (1 - w)
$$

### 9.3 Duration Estimation

$$
d_{\text{visit}} = d_{\text{base}} \times m_{\text{popularity}}, \quad m \in \{0.8, 1.0, 1.15, 1.3\}
$$

### 9.4 Cache Grid Resolution

$$
\text{gridKey} = \left(\lfloor \text{lat} \times 100 \rfloor / 100, \; \lfloor \text{lng} \times 100 \rfloor / 100\right) \approx 1.1 \text{ km resolution}
$$

### 9.5 Constraint Satisfaction

$$
\forall \text{ day}: \quad \sum t_{\text{travel}} + \sum d_{\text{visit}} \leq T_{\max} \quad \land \quad \sum d_{\text{haversine}} \leq D_{\max} \quad \land \quad h(a) \leq h_{\max} \quad \land \quad w(a) \leq w_{\max}
$$

---

*End of Report*
