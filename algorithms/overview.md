markdown
# Algorithmic Framework & Decision Engine Overview – Home Heating IoT
🇮🇹 *Per la versione in italiano, clicca [qui](overview.it.md).*

The centralized algorithms executed by the Python server on the Raspberry Pi represent the absolute "intelligence" of the system. They orchestrate and automate the following processes:

* ✅ **Environmental Telemetry:** Continuous multi-room internal and external temperature delta parsing.
* ✅ **Solar Yield Auditing:** Live monitoring of photovoltaic generation and storage matrix parameters.
* ✅ **Hybrid Thermal Integration:** Deterministic synchronization of the wood thermo-stove, heat pump, and distributed circulation loops.
* ✅ **Self-Consumption Maximization:** Secure, hardware-protected *Extra Heating* cycles.
* ✅ **Energy Conservation:** Elimination of transient ignition phases and short-cycling thermal waste.
* ✅ **System State Integrity:** Enforcing strict runtime alignment and validation checks across all edge devices.

Distributed microcontroller clients (ESP8266 and ESP32 edge arrays) are engineered to be lightweight sensing and interface nodes. All critical processing configurations and optimization updates are executed **exclusively by the central server node**, leveraging its computing power and access to years of historical relational data.

---

## 1. Algorithmic Modules

The platform isolates its home automation intelligence across multiple specialized software layers:

* **Extra Heating Optimizer:** Automatically redirects micro-generation capabilities based on live array calculations.
* **Biomass Priority Engine:** Monitors heat jacket curves to suppress heat pump operation when the secondary wood stove is firing.
* **HVAC Core Sync Engine:** Controls manifold timing cycles, pre-circulation sequences, and burner ignition variables.
* **Solar Matrix Interface:** Tracks real-time household electrical usage and battery state through the SolarEdge API.
* **Thermal Mass Analytics Layer:** Analyzes indoor climate profiles to isolate specific structural thermal decay rates.
* **Multi-Tier Safety Layer:** Hardened software constraints designed to shield mechanical plumbing circuits from thermal shock.

Every background process queries the flat, trigger-maintained row states inside the high-speed caching tables (`termostat_temp_now`, `external_temp_hum_now`, `termostat_warming_request`) while writing transaction records back into the historical ledgers.

---

## 2. Relational Interface Mapping

The automation processes consistently track parameters across the following persistent state models:

| Target Table | Strategic Purpose |
| :--- | :--- |
| `termostat_temp_now` | Queries immediate, current internal temperature values per room zone. |
| `termostat_warming_request` | Verifies active heating goals calculated by thermostat nodes. |
| `zone_status` | Audits the actual mechanical deployment position of physical electrovalves. |
| `heating_state` | Assesses and updates central plant power allocations. |
| `external_temp_hum_now` | Fetches live outdoor environmental variables to adjust heating start points. |
| `solar_now` | Parses instantaneous photovoltaic micro-generation capability data. |

---

## 3. Core Automation Decision Tree

```text
            ┌────────────────────────────────────────┐
            │       Thermostat State Evaluation      │
            └───────────────────┬────────────────────┘
                                │
                                ▼
            ┌────────────────────────────────────────┐
            │        Photovoltaic Output Audit       │
            └───────────────────┬────────────────────┘
                                │
                                ▼
            ┌────────────────────────────────────────┐
            │ Extra Heating Window Optimization Loop │
            └───────────────────┬────────────────────┘
                                │
                                ▼
            ┌────────────────────────────────────────┐
            │   Biomass Thermo-Stove Priority Check  │
            └───────────────────┬────────────────────┘
                                │
                                ▼
            ┌────────────────────────────────────────┐
            │   Central Plant (Boiler/Pump) Drive    │
            └───────────────────┬────────────────────┘
                                │
                                ▼
            ┌────────────────────────────────────────┐
            │    MariaDB State Table Updates Pushed  │
            └────────────────────────────────────────┘
```

---

## 4. Telemetry Sampling & Loop Frequencies

* **Extra Heating Engine:** Re-evaluates variables systematically every **5 minutes**.
* **Biomass Stove Monitor:** Instantaneous event-driven triggers capture loop temperature shifts.
* **HVAC Ignition Engine:** Runs a strict state validation check every **90 seconds**.
* **Database Cache Layer:** Real-time triggers inject fresh edge metrics instantly upon data insertion.
* **Meteorological Fetch:** Updates regional environmental forecasts every **10 minutes**.
* **SolarEdge Cloud Client:** Calls micro-generation parameters asynchronously every **5 minutes**.

---

## 5. System Safety & Operational Hardening

To guarantee hardware protection and maintain long-term reliability, the software stack enforces the following constraints:
* **Hysteresis Buffers:** Prevents rapid component toggling by applying precise thermal boundaries.
* **Structural Temperature Ceilings:** The *Extra Heating* function automatically drops specific zone actuators if local room telemetry breaks a strict **22°C threshold**.
* **Night Invalidation Rules:** Suppresses automated surplus optimization routines outside solar windows.
* **Biomass Compressor Interlocks:** Hardware-protective rules lock out heat pump activation during active thermo-stove operational cycles.
* **Edge Resilience Systems:** Distributed node firmware contains watchdogs that automatically trigger client device reboots if local WiFi connection drops out.
* **Transactional State Validation:** Cross-references database updates before modifying plant states to block invalid states.

---

## 6. Directory Document Structure

* **`extra_heating.md`:** Comprehensive layout of solar self-consumption surplus routing strategies.
* **`stove_logic.md`:** Hardware orchestration rules governing the biomass thermo-stove integration layer.
* **`adaptive_logic.md`:** Analytics models adjusting zone profiles based on thermal performance histories.
* **`diagrams.md`:** Deep-dive sequential logic trees mapping specialized mechanical state machines.
