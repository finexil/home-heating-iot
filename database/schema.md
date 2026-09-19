markdown
# Database Schema & Data Models – Home Heating IoT
🇮🇹 *Per la versione in italiano, clicca [qui](schema.it.md).*

The MariaDB database engine serves as the absolute logical core of the entire smart heating automation infrastructure.  
Distributed microcontrollers (ESP8266 and ESP32 edge nodes) execute targeted SQL operations against this schema to orchestrate ambient thermostats, manifold valve actuators, the centralized boiler plant, the hybrid biomass thermo-stove, and edge optimization algorithms.

The persistence layer partitions its domain into two distinct operational schemas:
* **`temperature` database:** Manages core runtime HVAC orchestration, manual user inputs, and live edge states.
* **`weather` database:** Processes regional meteorological conditions and forecast arrays.

This reference explicitly outlines the tables, data models, and fields contained within the primary `temperature` database container.

---

## 1. Table Definitions & Telemetry Models

### 1.1 `Warming_state`
Pushed by ambient thermostats to log discrete zone call-for-heat events and timeline transitions.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `id` | INT (AUTO_INCREMENT) | Primary key tracking the unique event index. |
| `room` | VARCHAR(50) | The specific room zone identifier (e.g., `"pt soggiorno"`). |
| `FLG_ON` | TINYINT(1) | Binary operational state flag: `0 = System Idle (OFF)`, `1 = Call for Heat (ON)`. |
| `timestamp` | DATETIME | The accurate server-side completion time of the event insertion. |

> ⚡ **Database Mechanism:** Pushing a new record to this table trips the `Update_termostat_warming_request` automated database trigger.

---

### 1.2 `termostat_warming_request`
A centralized lookup table caching the absolute latest target state required by each thermal zone. Engineered for fast, low-overhead edge retrieval.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `location` | VARCHAR(50) | System site tag. Constant value: `"ceresole"`. |
| `pt_soggiorno` | TINYINT(1) | Real-time state flag for Ground Floor Living Room/Kitchen. |
| `pt_camera` | TINYINT(1) | Real-time state flag for Ground Floor Bedroom. |
| `pt_bagno` | TINYINT(1) | Real-time state flag for Ground Floor Bathroom. |
| `p1_soggiorno` | TINYINT(1) | Real-time state flag for First Floor Living Room. |
| `p1_camera` | TINYINT(1) | Real-time state flag for First Floor Bedroom. |
| `p1_bagno` | TINYINT(1) | Real-time state flag for First Floor Bathroom. |
| `te_terrazzo` | TINYINT(1) | Real-time state flag for First Floor Balcony/Terrace loop. |

> 🎛️ **Hardware Polling target:** Distributed manifold zone controllers execute single-row `SELECT` loops against this table every 60 seconds to toggle hydraulic relays.

---

### 1.3 `termostat_temp_now`
A high-speed cache row storing immediate, real-time temperature telemetry to fuel rapid asynchronous updates for localized UI dashboard panels.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `location` | VARCHAR(50) | System site tag. Constant value: `"ceresole"`. |
| `pt soggiorno` | FLOAT | Current temperature reading for Ground Floor Living Room/Kitchen. |
| `pt camera` | FLOAT | Current temperature reading for Ground Floor Bedroom. |
| `pt bagno` | FLOAT | Current temperature reading for Ground Floor Bathroom. |
| `p1 soggiorno` | FLOAT | Current temperature reading for First Floor Living Room. |
| `p1 camera` | FLOAT | Current temperature reading for First Floor Bedroom. |
| `p1 bagno` | FLOAT | Current temperature reading for First Floor Bathroom. |
| `te terrazzo` | FLOAT | Current temperature reading for First Floor Balcony/Terrace loop. |

---

### 1.4 `termostat_temp_hum`
The main time-series ledger compiling long-term internal environments. Filled with structured 10-minute calculated rolling averages.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `id` | INT (AUTO_INCREMENT) | Primary key tracking the row sequence. |
| `room` | VARCHAR(50) | Target zone text identifier matching edge node hardware profiles. |
| `temperature` | FLOAT | Pushed 10-minute calculated localized temperature average value. |
| `humidity` | FLOAT | Pushed 10-minute calculated localized relative humidity average value. |
| `timestamp` | DATETIME | Data entry timeline point. |

> 📊 **Big Data Value:** Holds extensive historical data rows, supplying the Python predictive routines with raw building envelope data to isolate physical thermal decay traits.

---

### 1.5 `zone_status`
Updated directly by manifold actuator microcontrollers to register the absolute structural execution milestones of physical zone electrovalves.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `id` | INT (AUTO_INCREMENT) | Primary key tracking deployment events. |
| `room` | VARCHAR(50) | Associated room loop tracking manifold deployment targets. |
| `zone_on` | TINYINT(1) | Verified physical valve position: `0 = Closed`, `1 = fully Open`. |
| `timestamp` | DATETIME | Mechanical transition timestamp completion mark. |

---

### 1.6 `heating_state`
Monitors the global runtime state of the generation plant, thermal loop components, and active solar self-consumption optimization cycles.

| Field | Data Type | Description |
| :--- | :--- | :--- |
| `id` | INT (AUTO_INCREMENT) | Primary key tracking global plant transaction sequences. |
| `pump_on` | TINYINT(1) | Recirculation loop state flag: `0 = Inactive`, `1 = Pumping`. |
| `boiler_on` | TINYINT(1) | Core burner state flag: `0 = Standby`, `1 = Firing Heat Pump/Boiler`. |
| `stove_on` | TINYINT(1) | Wood Thermo-Stove status flag: `0 = Cold/Offline`, `1 = Active (Loop >= 60°C)`. |
| `timestamp` | DATETIME | System tracking checkpoint timestamp. |

---

### 1.7 `external_temp_hum` & `external_temp_hum_now`
Micro-climate sensors located outside the building wrapper provide structural boundary variables.
* **`external_temp_hum`:** Chronological time-series row appending data every 10 minutes.
* **`external_temp_hum_now`:** High-speed single-row lookup tracking real-time outside variables. Pushed every 5 seconds from a Wemos D1 Mini sensor cluster, instantly parsed using an event trigger loop.

---

## 2. Relational Logic Flow Summary

The data interactions across the core relational scheme operate under strict deterministic automation rules:

1. Distributed ambient room thermostats write call anomalies into `Warming_state` ──> A native MariaDB server trigger automatically reflects changes inside the `termostat_warming_request` cache.
2. Room thermostats write time-series rolling figures into `termostat_temp_hum` ──> A trigger automatically mirrors updates into the real-time layout of `termostat_temp_now`.
3. Edge manifold actuator nodes poll the cached states inside `termostat_warming_request` every 60 seconds to switch hydraulic loops.
4. Manifold units declare mechanical valve execution completions by modifying fields within `zone_status`.
5. The global boiler logic engine acts upon the aggregated requirements computed inside `zone_status` and `heating_state`.
6. Advanced analytical background automation scripts (SolarEdge API tracking, Extra Heating rules) dynamically re-arrange system constraints inside `heating_state`.
