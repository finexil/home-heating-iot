markdown
# Home Heating IoT System

🇮🇹 *Per la versione in italiano, clicca [qui](README.it.md).*

An advanced, data-driven DIY Home Automation system powered by **ESP8266, ESP32, and Raspberry Pi** to manage complex, hybrid domestic heating infrastructures.

## 📌 Project Overview
Managing a modern hybrid heating system requires orchestrating multiple thermal sources, electronic components, and real-time automation algorithms. This completely cloud-free, **100% local IoT ecosystem** is designed to precisely control a complex multi-zone residential installation:

* **Hydronic Radiant Floor Heating** distributed across multiple independent zones.
* **Primary Heat Pump** and secondary electric boiler integration.
* **Biomass Hybrid Source:** Wood-burning thermo-stove integration with automatic safety bypass.
* **Photovoltaic Array with Storage:** 7kWh battery integration for automated energy redirection.

### Key Capabilities
* **Granular Monitoring:** Localized ambient temperature, humidity, CO₂, and TVOC sampling.
* **Smart Manifold Control:** Driven by low-power edge nodes with protective mechanical safety timeouts.
* **Self-Consumption Maximization:** Custom *Extra Heating* algorithms driven by real-time solar surplus calculations.
* **Big Data Edge Analytics:** Years of historical micro-climate logs stored locally for thermal inertia modeling.

## 🏗️ System Architecture
The layout isolates edge sensing and actuation from the central data persistence layer across three main hardware tiers:

┌───────────────────────────┐        ┌──────────────────────────┐         ┌───────────────────────────────┐
│       Nodi Periferici     │ ─────> │      Database Locale     │  ─────> │      Algoritmi Analitici      │
│      (ESP8266 / ESP32)    │        │       (Raspberry Pi)     │         │     (Surplus / Previsioni)    │
└───────────────────────────┘        └──────────────────────────┘         └───────────────────────────────┘

1. **Distributed Thermostats (ESP8266 & ESP32):** Edge processing nodes handling human-machine interface (HMI), sensor sampling, and localized control loops.
2. **Zone & Generation Controllers (ESP8266):** Actuator units driving manifold electrovalves, boiler relays, and pump circulation circuits.
3. **Central Automation Engine (Raspberry Pi 2B):** Local Linux node running **MariaDB**, serving telemetry dashboards, and executing scheduling/optimization scripts.

## 🌡️ Edge Nodes: Hardware & Logic Overview

The project leverages two hardware form factors tailored for specific room requirements:

### 🔹 Compact Thermostats (ESP8266 + 1.8" SPI Display)
Deployed in secondary bedrooms and bathrooms. Utilizes physical buttons for direct navigation and manual overrides.

### 🔹 Advanced Touch Nodes (ESP32 + 3.5" TFT Touchscreen)
Deployed in open-plan living areas. Features premium graphical monitoring and extended environment analytics.

|          Feature / Metric         | Compact Node (ESP8266) | Touch Node (ESP32) |
| --------------------------------- | -----------------------| ------------------ |
|     **Temperature & Humidity**    |           ✅           |         ✅        |
|     **24-Hour Graphical Trend**   |           ✅           |         ✅        |
|     **Manual Target Overrides**   |           ✅           |         ✅        |
|     **CO₂ & TVOC Air Quality**    |           ❌           |         ✅        |
| **6-Hour Local Weather Forecast** |           ❌           |         ✅        |

### 🔥 Intelligent Thermal Source Indicators
The graphical user interface dynamically reflects the active power source feeding the hydronic loops through system-wide status flags:
* **Red Flame:** Standard active heating requested by local room schedule rules (`WARM_STATE`).
* **Yellow Flame:** *Extra Heating* cycle activated due to solar production surplus.
* **Stove Icon:** Thermal power supplied entirely by the active wood thermo-stove system.

> ⚠️ **Overheating Protection Rule:** Both *Extra Heating* and *Thermo-Stove* modes force global zone activation but automatically trip open to cut flow if individual rooms cross a strict **22°C threshold**.

### 🔧 Edge RTOS Task Architecture
To prevent UI freezing, ensure network reliability, and guarantee deterministic behavior during heavy sensor querying, the ESP32 touch nodes utilize **FreeRTOS** split into prioritized task loops:
* `Task_Sensors` (High Priority) — Continuous micro-second polling of environment metrics.
* `Task_Logic` (Medium Priority) — Hysteresis calculation and room state determination.
* `Task_Display` (Medium Priority) — Pushes screen buffer redraws only when state shifts occur.
* `Task_Network` (Low Priority) — Asynchronous WiFi calls and MariaDB remote transactions.

## 🎛️ Manifold Valve & Boiler Automation

### Actuator Edge Nodes (ESP8266 Lolin D1 Mini Lite)
* Synchronizes against database transaction state tables every 60 seconds.
* Compensates for slow thermal manifold components (approx. **6-minute opening/closing mechanical latency**).
* Applies soft start-up rules to shield active system infrastructure from sudden system pressure shifts.

### Generation Controller (ESP12-E)
* Interlocks heat pump operation against real-time biomass thermo-stove status.
* Executes **90-second pre-circulation pump sequences** before heating ignition to protect internal boiler exchangers.
* Eliminates short-cycling behavior through software-hardened dampening constraints.

## 💾 Core Infrastructure: Raspberry Pi 2B

### 🗄️ MariaDB Telemetry Database
Stores runtime operational metrics, user profile tables, and raw time-series telemetry logs.
* **SQL Trigger Layer:** Automates live calculations upon transactional entries to build dynamic summary tables (`termostat_warming_request`, `external_temp_hum_now`).
* **Performance Gain:** Triggers vastly reduce database processing strain and network latency during concurrent client polling cycles.

### 🌐 Local Analytical Dashboard
A private, non-exposed local environment providing comprehensive analytical insight:
* Operational runtime charts across all independent thermal zone manifolds.
* Building envelope response maps tracking historical indoor conditions against OpenWeather data.
* Micro-climate behavior charting and electrical consumption metrics.

## 🧠 Optimization: The Extra Heating Algorithm

The primary background routine continuously parses home power variables to optimize solar self-consumption:

[Solar Array Production] ──> [Battery State >= Threshold] ──> [Trigger Extra Heating]│(Global Zone Drive under 22°C)
1. Queries live photovoltaic inverter values and the **7kWh battery bank** status metrics between **09:30 and 16:30**.
2. If excess generation is detected alongside complete battery recovery, the algorithm shifts the home into an **Extra Heating state (Yellow Flame)**.
3. Automatically turns structural floor thermal mass into a clean energy accumulator, mitigating grid-feed issues and lowering night consumption requirements.

## 🛠️ Main Engineering Challenges Solved
* **Distributed Synchronization:** Designed crash-resilient edge code to safely align standalone MCU arrays with SQL tables without collision hazards.
* **Thermal Inertia Damping:** Developed calculation models to counteract the significant lag properties inherent in hydronic slab installations.
* **HMI Performance Tuning:** Architected concurrent FreeRTOS task models on the ESP32 platform to completely isolate layout drawing lags from critical system network calls.
* **Cloud Independence:** Implemented a robust 100% edge topology maintaining high availability during complete external internet connection dropouts.

## 📂 License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file 
