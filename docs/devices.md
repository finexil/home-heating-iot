# Technical Configuration of System Hardware Nodes – Home Heating IoT

🇮🇹 *Per la versione in italiano, clicca [qui](devices.it.md).*

This document delivers an in-depth hardware and architectural analysis of the core physical equipment and embedded systems constituting the Home Heating IoT platform: ambient thermostats, manifold controllers, boiler/heat pump relays, biomass interfaces, atmospheric sensory grids, and the centralized edge database engine.

It serves as a comprehensive reference manual detailing the electrical routing configurations, firmware tasks, and topological constraints of the hardware network.

---

## 1. Ambient Thermostat Nodes (ESP8266 & ESP32)

Thermostat units act as the primary human-machine interfaces (HMI) for the home's heating grid. They query local environment metrics, evaluate active scheduling tables, push data vectors to the persistence layer, and display systemic diagnostic graphics.

### 1.1 Hardware Specifications

#### 🔹 Compact Micro-Controller Node (ESP8266 – 12E array)
* Deployed inside secondary bedrooms and high-humidity bathroom spaces.
* Integrates a low-power 1.8" SPI display matrix managed via physical tactile button arrays.
* Monitors localized relative humidity and room temperature indicators.
* Displays 24-hour micro-climate historical trend charts.
* Supports active multi-source visual context feedback icons.
* Commits calculated rolling data frames to the database engine exactly every 10 minutes.

#### 🔹 Premium Touch Management Terminal (ESP32 SoC)
* Deployed inside main living room areas, open kitchens, and balcony loops.
* Interfaces via a high-density 3.5" TFT touchscreen module.
* Features integrated gas sampling modules monitoring CO₂ and TVOC indicators.
* Fetches and maps local 6-hour predictive weather forecast payloads.
* Provides a systemic, multi-floor zone status grid monitoring the whole building wrapper.
* Fluid, high-availability graphical user interface driven by native FreeRTOS thread splits.

---

### 1.2 Connected Sensor Arrays
Peripherals interface directly with the edge microcontrollers via digital GPIO lines utilizing pull-up resistor constraints where appropriate:
* **DHT22** — Digital relative humidity and temperature sensor arrays (deployed across legacy ESP8266 client units).
* **BME280** — Environmental sensor tracking barometric pressure, relative humidity, and ambient room temperature.
* **MQ-135 Sensor Module** — Gas sensor array parsing interior air quality metrics (specifically mapping carbon dioxide gas and total volatile organic compounds).

---

### 1.3 Control Loop Directives: `WARM_STATE` and `WARM_FORCE`
To calculate the true real-time heating requirement profile, each thermostat client evaluates two distinct operational variables:
* **`WARM_STATE`** — A boolean parameter calculated against the active weekly scheduler matrix defining if the current hours require thermal delivery.
* **`WARM_FORCE`** — An explicit, user-asserted override byte field that changes operational rules:
  * `0 = Normal` (Firmware ignores parameters and tracks the automated `WARM_STATE` scheduler curve).
  * `1 = Forced ON` (Overrides schedule blocks and demands immediate hot water delivery).
  * `2 = Forced OFF` (Locks out the heating loops entirely, ignoring temperature triggers).

The combination of these parameters establishes the target configuration value committed directly to the central data layer.

---

### 1.4 User Interface States & Icon Mapping
The screen buffer updates dynamic graphical icons to communicate the active thermodynamic source supplying energy into the zone:

* <img width="40" height="39" alt="image" src="https://github.com/user-attachments/assets/d97f6b0c-e857-4e03-8112-6adf18e54451" /> **Red Flame** — Standard local room heat request calculated against local scheduler tables.
* <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/6389740f-86ea-4227-b1d1-031ebf1e72d3" /> **Yellow Flame** — Automated *Extra Heating* sequence driven by real-time solar photovoltaic net surplus.
* <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/53d6b121-8ec9-4bc0-9891-884fe79045de" /> **Thermo-Stove** — Hot water loops supplied entirely by active biomass fuel sources.

> ⚠️ **Overheating Protection Constraints:** Both *Extra Heating* and *Thermo-Stove* configurations force global hydraulic deployment. However, the localized edge firmware automatically blocks flow delivery to individual spaces if ambient room readings cross a strict **22°C ceiling**.

---

### 1.5 Edge Processing Multithreading via FreeRTOS (ESP32)
To secure absolute system responsiveness and prevent graphical lagging or packet loss during heavy network blocking events, the ESP32 touch code base splits code loops into prioritized FreeRTOS task stacks:
* `Task_Sensors` (High Priority) — Continuous high-frequency querying of ambient metrics and digital lines.
* `Task_Logic` (Medium Priority) — Automated evaluation of thermal hysteresis calculations.
* `Task_Display` (Medium Priority) — Handles screen redraw requests only when telemetry shifts occur.
* `Task_Network` (Low Priority) — Non-blocking, asynchronous WiFi handling and MariaDB SQL transactions.

---

## 2. Hydraulic Manifold Controllers (ESP8266 Lolin D1 Mini Lite)

These edge actuation stations physically orchestrate flow delivery by toggling the low-voltage circuitry driving the 230V mechanical electrovalves of the hydronic floor slabs.

### 2.1 Technical Responsibilities
* Polls the centralized relational data persistence tier (parsing single-row `zone_status` lookup caches) every 60 seconds.
* Switches high-voltage 230V manifold electrovalve lines via isolated transistor-to-relay breakout networks.
* Applies protective **90-second mechanical lag adjustments** matching the slow mechanical opening/closing latency of individual zone valves.
* Updates verified mechanical state feedback parameters back into the relational schema.
* Orchestrates 2 to 4 independent thermal zones per physical controller board layout.

---

## 3. Central Plant Generation Controller

Coordinates primary heating elements, protects central exchangers, and manages hybrid biomass transitions.

### ⚙️ Automation Ignition Sequence:
1. When at least one standalone room zone reports a verified active heat call inside `zone_status` ──> Instantly triggers the high-volume primary manifold recirculation distribution pump.
2. Executes a strict **30-second pre-circulation protective timeout loop** ──> Safely drops in the primary electric heat pump/boiler burner relays.
3. If the biomass thermo-stove subsystem transitions to active ──> Keeps the secondary heat exchanger systems available, allowing the plant to detect the high-temperature water mass and drop down compressor operations.
4. Shut-down cycle sequence ──> Instantly drops out the burner element lines while keeping the distribution pumps running for an extended duration to flush residual thermal mass out of the plant core.

---

## 4. Biomass Hybrid Thermo-Stove Monitor Node

The secondary solid-fuel wood thermo-stove interfaces with the heating ecosystem via a dedicated isolation layer:
* High-volume secondary plate heat exchanger.
* Dedicated high-temperature circulation pump assembly.
* Analogue water jacket immersion sensor arrays.

Whenever the embedded edge sensor registers water jacket metrics crossing the **60°C target**, it trips a physical loop relay:
* The local node forces the circulation pump into an active state.
* Commits a priority status update to the database layer, changing global system parameters to biomass mode (**Stove** icon shown globally).
* Extends thermal flow across all floor radiant manifolds under the strict protection of the 22°C safety limits.

---

## 5. Outdoor Meteorological Station (ESP8266 Wemos D1 Mini Pro)

A weather-proof edge sensing array that samples atmospheric criteria every 5 seconds:
* True ambient outdoor temperature curves.
* Atmospheric relative humidity percentages.
* Local barometric pressure indices.

The peripheral micro-controller computes a **10-minute mathematical rolling average** and inserts the telemetry dataset directly into the database engine. These metrics are actively parsed to drive predictive thermal mass calculations, manage solar *Extra Heating* thresholds, and render weather visualizations across internal panels.

---

## 6. Network Communications Stack

Every distributed microcontroller asset interacts across a unified local architecture:
* A private, non-exposed local **Wi-Fi infrastructure**.
* Flat, targeted SQL query primitives executed via light database drivers.
* High-speed single-row **`*_now` cache views** designed to prevent loop execution blocks.
* Automated server-side database **triggers** that update cache contexts immediately upon telemetry shifts.

> 🔒 No commercial cloud dependencies or external third-party servers are utilized anywhere within the framework.

---

## 7. Raspberry Pi 2B Central Server

### Software Package Roles:
* **MariaDB Server Engine:** Handles transactional data, runtime states, configurations, and long-term storage logs.
* **Local Web Interface:** Serves internal dashboard visualization tools across the secure home network.
* **Python 3 Automation Routines:** Executes continuous background scheduling, predictive modeling, and analytics.
* **Photovoltaic Client Application:** Interfaces asynchronously with the SolarEdge Cloud API every 5 minutes to fetch micro-generation and storage metrics.

---

## 8. Core Automation Modules

### 🔶 Solar Extra Heating Engine
Directs clean photovoltaic energy surplus into the residential floor mass inside the daily high-irradiance execution window.

### 🔶 Predictive Adaptive Logic
Analyzes internal temperature delta vectors against outdoor micro-climates and historical envelope response traits to calculate exact radiant slab thermal latency metrics.

### 🔶 Biomass Priority SystemOrchestrates high-temperature thermal delivery from the wood-burning stove while completely locking out electric compressors to ensure zero grid utility cost.

## 9. Structural Hardware Documentation IndexThe following specific technical schematic manuals describe the electrical layouts summarized in this manual:Comprehensive hardware wiring charts and microcontroller pinout assignments.Physical circuit schematics detailing the low-power transistor-driven relay arrays.Core SQL database DDL files and production-ready server trigger configurations.High-level sequence trees mapping systemic data transactions.
