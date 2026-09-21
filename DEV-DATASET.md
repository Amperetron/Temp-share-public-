## Attributes:

| Feature Name | Type | Description / Logic |
|---|---|---|
| entity_id | String | Unique ID (WTR-000001 to WTR-020000) |
| timestamp | Datetime | Measurement date (spanning across seasons) |
| property_type | Categorical | Residential, Multi-Family, Commercial, Industrial, Agricultural |
| occupants_or_workforce | Integer | Headcount (1–8 for residential; 10–250 for commercial) |
| area_sqft | Float | Property / facility footprint |
| has_pool | Boolean | Presence of swimming pool / water feature |
| lawn_area_sqft | Float | Irrigated landscaping footprint |
| climate_zone | Categorical | Arid, Semi-Arid, Mediterranean, Continental, Tropical |
| avg_temperature_c | Float | Seasonal ambient temperature |
| daily_consumption_liters | Float | Target metric: calculated based on baseline + climate - conservation |
| leak_detected | Boolean | High-consumption anomaly flag (~3–5% of records) |
| smart_meter_installed | Boolean | IoT infrastructure flag |
| conservation_measures | Multi-label | Rainwater harvesting, Greywater reuse, Drip irrigation, Low-flow aerators, None |
| water_source | Categorical | Municipal Grid, Groundwater Well, Desalination, Recycled Line |
| water_cost_usd | Float | Billed cost modeled via tiered pricing tariffs |


## code for 20000 entities: 

```
import numpy as np
import pandas as pd

# Set seed for reproducible synthetic generation
np.random.seed(42)
N = 20_000

# 1. Base Identifiers and Dates
entity_ids = [f"WTR-{i:06d}" for i in range(1, N + 1)]
dates = pd.date_range(start="2024-01-01", end="2025-12-31", periods=N)

# 2. Property Classifications
property_types = np.random.choice(
    ["Single-Family", "Multi-Family", "Commercial", "Industrial", "Agricultural"],
    size=N,
    p=[0.55, 0.20, 0.15, 0.05, 0.05],
)

climate_zones = np.random.choice(
    ["Arid", "Semi-Arid", "Mediterranean", "Continental", "Tropical"],
    size=N,
    p=[0.25, 0.25, 0.20, 0.20, 0.10],
)

water_sources = np.random.choice(
    ["Municipal Grid", "Groundwater Well", "Recycled Utility Line", "Mixed"],
    size=N,
    p=[0.70, 0.15, 0.10, 0.05],
)

smart_meter_installed = np.random.choice([True, False], size=N, p=[0.65, 0.35])

# Conservation measures applied
measures_pool = [
    "None",
    "Low-Flow Fixtures",
    "Rainwater Harvesting",
    "Drip Irrigation",
    "Greywater Recycling",
    "Dual-Flush & Aerators",
]
conservation_applied = np.random.choice(
    measures_pool, size=N, p=[0.25, 0.25, 0.15, 0.15, 0.10, 0.10]
)

# 3. Structural Attributes by Property Type
occupants = []
area_sqft = []
lawn_sqft = []
has_pool = []

for p in property_types:
    if p == "Single-Family":
        occ = np.random.randint(1, 6)
        area = np.random.uniform(900, 4500)
        lawn = np.random.uniform(100, 3000) if np.random.rand() > 0.3 else 0.0
        pool = np.random.choice([True, False], p=[0.18, 0.82])
    elif p == "Multi-Family":
        occ = np.random.randint(8, 60)
        area = np.random.uniform(4000, 25000)
        lawn = np.random.uniform(0, 1500) if np.random.rand() > 0.6 else 0.0
        pool = np.random.choice([True, False], p=[0.12, 0.88])
    elif p == "Commercial":
        occ = np.random.randint(15, 200)
        area = np.random.uniform(5000, 60000)
        lawn = np.random.uniform(0, 2000) if np.random.rand() > 0.7 else 0.0
        pool = False
    elif p == "Industrial":
        occ = np.random.randint(30, 400)
        area = np.random.uniform(20000, 150000)
        lawn = 0.0
        pool = False
    else:  # Agricultural
        occ = np.random.randint(2, 15)
        area = np.random.uniform(50000, 500000)
        lawn = np.random.uniform(10000, 100000)
        pool = False

    occupants.append(occ)
    area_sqft.append(round(area, 1))
    lawn_sqft.append(round(lawn, 1))
    has_pool.append(pool)

# 4. Environmental & Consumption Logic (in Liters/Day)
# Standard indoor consumption: ~120 - 150 L per person/day
# Outdoor irrigation: ~5 - 10 L per sqft of lawn
# Pools: ~200 - 400 L/day baseline maintenance & evaporation
base_indoor = np.array(occupants) * np.random.normal(135, 15, size=N)
lawn_demand = (np.array(lawn_sqft) * np.random.uniform(1.2, 3.5, size=N)) / 7.0
pool_demand = np.array([300.0 if p else 0.0 for p in has_pool])

# Climate multipliers
climate_multipliers = {
    "Arid": 1.35,
    "Semi-Arid": 1.20,
    "Mediterranean": 1.10,
    "Continental": 1.00,
    "Tropical": 0.90,
}
mult_array = np.array([climate_multipliers[c] for c in climate_zones])

# Conservation offsets (reduction percentage)
conservation_discounts = {
    "None": 1.00,
    "Low-Flow Fixtures": 0.85,
    "Rainwater Harvesting": 0.80,
    "Drip Irrigation": 0.82,
    "Greywater Recycling": 0.75,
    "Dual-Flush & Aerators": 0.88,
}
disc_array = np.array([conservation_discounts[m] for m in conservation_applied])

# Anomaly / Leak flag (~4% rate)
leak_flags = np.random.choice([0, 1], size=N, p=[0.96, 0.04])
leak_volume = leak_flags * np.random.uniform(500, 3000, size=N)

# Final daily consumption calculation
daily_liters = (
    (base_indoor + (lawn_demand + pool_demand) * mult_array) * disc_array
) + leak_volume
daily_liters = np.clip(daily_liters, 80, None)  # min bound

# Calculate estimated cost (Tiered: baseline $0.002/L up to 1500L, then $0.0035/L)
tier_base = np.minimum(daily_liters, 1500) * 0.002
tier_excess = np.maximum(0, daily_liters - 1500) * 0.0035
daily_cost = np.round(tier_base + tier_excess, 2)

# 5. Assemble DataFrame
df = pd.DataFrame(
    {
        "entity_id": entity_ids,
        "record_date": dates.strftime("%Y-%m-%d"),
        "property_type": property_types,
        "climate_zone": climate_zones,
        "occupants_or_staff": occupants,
        "property_area_sqft": area_sqft,
        "irrigated_lawn_sqft": lawn_sqft,
        "has_pool": has_pool,
        "smart_meter_enabled": smart_meter_installed,
        "conservation_measure": conservation_applied,
        "water_source": water_sources,
        "leak_detected": leak_flags.astype(bool),
        "daily_consumption_liters": np.round(daily_liters, 1),
        "daily_cost_usd": daily_cost,
    }
)

# Export to CSV
df.to_csv("water_consumption_20k.csv", index=False)
print(f"Generated {len(df)} entities successfully into 'water_consumption_20k.csv'")
```
