markdown
# Database Architecture Overview – Home Heating IoT
🇮🇹 *Per la versione in italiano, clicca [qui](overview.it.md).*

This document provides a comprehensive structural overview of the MariaDB database engine powering the Home Heating IoT ecosystem.  
It serves as the core architectural baseline that glues together the following technical references:

* **[schema.md](schema.md):** Detailed table schemas and data models.
* **[triggers.md](triggers.md):** Server-side trigger functions and deployment scripts.
* **devices.md:** Device-specific documentation on database integration.

The objective of this guide is to explain **how telemetry data flows** through the network and **why the database serves as the absolute heart of the entire automation system**.

## 1. High-Level Database Topology

The ecosystem segregates its domain operations into two distinct database containers:

* **`temperature`:** Handles runtime heating logic, thermostat state checks, and physical zone actuation tracking.
* **`weather`:** Manages external micro-climate analytics and multi-hour forecasts.

Distributed hardware edge nodes (Thermostats, Actuators, and Boiler Controllers) exclusively read and write to the `temperature` database. The central Python automation routine running on the Raspberry Pi actively interfaces with both environments.

### Core Architectural Design Pillars
* ✅ **Minimize Edge Load:** Strips relational data processing overhead away from low-power ESP8266 and ESP32 nodes.
* ✅ **Zero-Latency Injections:** Utilizes event-driven, server-side triggers for immediate configuration mirroring.
* ✅ **Flat Row Read Execution:** Delivers fully processed, instantly readable metrics to edge devices.
* ✅ **Chronological Data Auditing:** Preserves extensive multi-year historical logs to feed adaptive learning algorithms.

## 2. Table Classification & Structural Roles

### 2.1 The "Time-Series" Auditing Layer
These tables hold growing datasets designed for longitudinal analysis:
* `Warming_state` — Logs chronological thermostat state evaluations and binary target events (ON/OFF).
* `termostat_temp_hum` — Tracks indoor environmental telemetry sampled at systematic 10-minute intervals.
* `external_temp_hum` — Logs outdoor barometric pressure, humidity, and temperature trends every 10 minutes.
* `zone_status` — Records physical actuator deployment milestones (inserted exclusively during mechanical shifts).
* `heating_state` — Logs central plant power transactions (Heat Pump, Auxiliary Boiler, and Recirculation loops).

**System Value Provided:**
* ✅ Long-term structural thermal performance reports.
* ✅ Data-rich dashboard visualizations.
* ✅ Adaptive algorithms built on real-world building envelope metrics.

### 2.2 The "NOW" High-Speed Caching Layer
Single-row cache lookups engineered for **instant, single-operation retrieval**:
* `termostat_warming_request`
* `termostat_temp_now`
* `external_temp_hum_now`

**System Value Provided:**
* ✅ ESP8266 and ESP32 edge clients fetch their active loop goals by querying a single row.
* ✅ Eliminates slow `JOIN` commands, matching queries, or computational load at the edge.
* ✅ Secures extreme system stability when operating over intermittent local WiFi paths.

> ⚡ **Automation Mechanic:** The single-row tracking metrics in the "NOW" cache tables are entirely maintained by automated database **triggers**.

## 3. System Data Flow Components

The system lifecycle operates across 4 independent processing loops interacting with the persistence engine.

### 3.1 The Thermostat Loop (Edge Sensors → DB)
Each independent thermostat node executes the following routine:
1. Samples local ambient temperature and relative humidity variables every 5 seconds.
2. Computes a rolling metric average every 10 minutes and inserts it into the `termostat_temp_hum` time-series data table.
3. Upon detecting a schedule state shift, instantly inserts a state row into the `Warming_state` log.
4. Periodically updates its local runtime properties by calling current configuration rules from the database.

**Trigger Side-Effects:**
* `Warming_state` updates trigger automatic updates to `termostat_warming_request`.
* `termostat_temp_hum` insertions populate current data directly into `termostat_temp_now`.

> 🚫 **Decoupled Architecture:** Thermostat nodes **NEVER** communicate directly with manifold actuators or central boilers. All operations are mediated by database state tables.

### 3.2 The Actuator Loop (DB → Physical Manifolds)
Each distributed zone valve controller executes the following logic:
1. Polls the flat rows of the `termostat_warming_request` table every 60 seconds.
2. Directs onboard electrical relays to actuate corresponding hydraulic zone valves.
3. Confirms mechanical transition success by updating the true status field inside `zone_status`.

> 💡 **Edge Simplicity:** Manifold valve controllers execute zero processing math and maintain no state calculations locally.

### 3.3 The Generation Loop (Boiler / Thermo-Stove → DB)
The centralized primary **Boiler Controller** runs the following automation sequence:
1. Assesses the system wide load requirements by reading the `zone_status` data matrix.
2. References active safety overrides inside `heating_state`.
3. Evaluates hardware safety delay thresholds and executes burner ignition or pump circulation routines.
4. Commits active engine milestones back into the `heating_state` ledger.

The biomass **Thermo-Stove Controller** integrates via a standalone loop:
1. Detects water loop temperature reaching the system-wide threshold (60°C).
2. Updates global energy parameters inside `heating_state`.
3. Forces global heating zone enablement across all independent loops (overridden only if individual spaces exceed **22°C**).

### 3.4 The Optimization Loop (Python Core Engine → DB)
The centralized Python scripts running on the Raspberry Pi core perform deep database parsing:
* Continuously queries live indoor values from the `termostat_temp_now` cache view.
* Pulls instantaneous outdoor trend vectors from the `external_temp_hum_now` table.
* Evaluates electrical variables and array production capabilities using real-time API queries.
* Updates `heating_state` flags to systematically trigger solar self-consumption optimization (**Extra Heating**).
* Calculates and injects adaptive hourly operational updates into the zone profile configurations.


## 4. System Topology & Data Flow Map

```text
            ┌────────────────────┐
            │     Termostats     │
            │ ESP8266 / ESP32    │
            └─────────┬──────────┘
                      │ INSERT
                      ▼
            ┌───────────────────┐
            │   Warming_state   │──────┐
            └───────────────────┘      │ trigger
                                       ▼
                        ┌────────────────────────────┐
                        │ termostat_warming_request  │
                        └────────────────────────────┘
                                       │
                                       ▼
                        ┌────────────────────────────┐
                        │  Solenoid valve actuators  │
                        └─────────────┬──────────────┘
                                      │ UPDATE
                                      ▼
                           ┌──────────────────────┐
                           │     zone_status      │
                           └──────────┬───────────┘
                                      │
                                      ▼
                          ┌─────────────────────────┐
                          │   Boiler / wood Stove   │
                          └───────────┬─────────────┘
                                      │ WRITE
                                      ▼
                          ┌─────────────────────────┐
                          │     heating_state       │
                          └───────────┬─────────────┘
                                      │
                                      ▼
                       ┌───────────────────────────────┐
                       │    Raspberry Pi Algorithms    │
                       └───────────────────────────────┘
```

## 5. Architectural Design Rationale

### Centralized Relational Persistence Benefits
Choosing a central MariaDB architecture over standard peer-to-peer frameworks ensures:
* Automated, crash-resilient runtime state alignment across all distributed embedded assets.
* Complex predictive analytics executions completed without shifting calculation burden to edge microcontrollers.
* Complete multi-year time-series historical storage for advanced thermal mass degradation audits.
* 100% data sovereignty and system availability during total external internet connection outages.

### Why not MQTT, Home Assistant, or Node-RED?
While generic home automation middleware abstractions are viable alternative solutions for standard consumer homes, this hybrid infrastructure demands:
* Millisecond-accurate hydraulic synchronization across distinct multi-floor manifolds.
* Explicitly deterministic automation handling to protect mechanical components against short-cycling fatigue.
* Server-side database transactions providing an audit-ready telemetry ledger.

A native database layer natively satisfies these rigorous reliability properties without external dependencies.

## 6. Document Matrix References

This overview manual frames the persistence strategy of the platform. To dig deeper into the physical implementations, reference the following specific guides:

* **[schema.md](schema.md):** Deep breakdown of field properties, data constraints, and system parameters.
* **[triggers.md](triggers.md):** Deployment definitions and raw SQL trigger definitions.
* **devices.md:** Hardware execution logs explaining node network interaction patterns.

