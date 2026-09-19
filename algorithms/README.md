
# Automation Algorithms & Predictive Logic

This directory documents the mathematical frameworks, state machines, and predictive background routines executed by the central Python 3 automation engine on the Raspberry Pi. 

These routines process real-time telemetry, local storage persistence tables, and external API inputs to dynamically maximize energy efficiency and manage building thermal mass.

---

## 📌 Core Architectural Systems

'''text
          [SolarEdge API]          [MariaDB Caching]         [OpenWeather API]                        
                 │                         │                         │
                 ▼                         ▼                         ▼
      ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
      │ Extra Heating Engine │  │ Biomass Hybrid Logic │  │ Thermal Inertia Map  │
      │(Solar Self-Consump.) │  │(Water Loop Overrides)│  │(Envelope Predictor)  │
      └──────────────────────┘  └──────────────────────┘  └──────────────────────┘
'''
The system automation behavior is driven by three decoupled core logical modules:

### 1. The "Extra Heating" Engine (Solar Self-Consumption)
Designed to systematically redirect micro-generation surplus into the physical home framework, acting as a virtual energy buffer.
* **API Integration:** Connects directly to the SolarEdge API endpoint to monitor real-time inverter photovoltaic output, household load curves, and the state of charge (SoC) of the **7kWh storage battery bank**.
* **Temporal Window:** Restricts evaluation tracking to the **09:30 – 16:30** daily time-frame to guarantee reliable solar irradiance conditions even through winter cycles.
* **Thermal Injection:** If generation surplus is detected and the battery bank has recovered, the algorithm commands global system overrides (`heating_state`), forcing climate zones into an active state (**Yellow Flame** icon on thermostats) to turn the hydronic slab mass into a thermal accumulator.
* **Hysteresis Boundary:** Includes strict ambient safety ceilings; optimization overrides instantly trip open for any specific zone if its temperature passes a hard **22°C limit**.

### 2. Hybrid Biomass & Flow Stabilization Logic
Orchestrates the interplay between the primary electric heat pump and the secondary wood-burning thermo-stove system.
* **Biomass Loop Tracking:** Edge inputs watch the thermo-stove water jacket temperature. When the loop meets the **60°C stabilization threshold**, the controller closes a physical relay to enable the primary 220V circulation loop.
* **Mechanical Lag Compensation:** Hydronic floor radiant manifolds exhibit distinct opening and closing delays (approx. **6 minutes**). To prevent hardware stress and pump cavitation, the control loop inserts a structured **90-second pre-circulation sequence** before firing heating elements.
* **Staged Staggered Enablement:** When the biomass stove is active, it suppresses heat pump compressor engagement while implementing a staggered zone timing algorithm to evenly balance the cold return flow:
  * **T + 7 mins:** Bathrooms (Ground Floor / First Floor)
  * **T + 15 mins:** Bedrooms (Ground Floor / First Floor)
  * **T + 25 mins:** First Floor Living Room
  * **T + 30 mins:** Ground Floor Living Room
  * **T + 40 mins:** Terrace loop

### 3. Predictive Thermal Inertia Mapping
A data-driven building envelope analysis tool that counteracts the significant structural delays inherent in floor mass radiant setups.
* **Telemetry Ledger Analysis:** Continuously evaluates over **24 months of historical environment telemetry** logged at 10-minute intervals (`powerDetails` and `External_temp_hum`).
* **Boundary Context Cross-Referencing:** Pairs localized room temperature delta fluctuations (\(\Delta T = T_{t} - T_{t-1}\)) against 6-hour predictive regional forecast updates fetched every 3 hours from the OpenWeather API (`weather` database).
* **Predictive Pre-Heating:** Instead of operating under fixed hourly schedules, the central engine computes the unique degradation speed of individual rooms relative to outdoor conditions, dynamically advancing or delaying hydronic valve open commands to achieve target parameters exactly on time.

---

## 📂 Upcoming Implementation Documentation
* **`extra_heating.py`**: Production script definitions for handling SolarEdge authentication, payload parsing, and runtime table mutations.
* **`thermal_models.ipynb`**: Data science scripts plotting temperature delta counts across the six primary indoor living zones under varying outdoor weather conditions.
