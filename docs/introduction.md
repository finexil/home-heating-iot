# Technical Introduction – Home Heating IoT Architecture

🇮🇹 *Per la versione in italiano, clicca [qui](introduction.it.md).*

This section provides a comprehensive technical overview of the distributed IoT architecture engineered to manage a complex domestic heating system using ESP8266, ESP32, and Raspberry Pi hardware layers.

The documentation compiled here complements the general introduction provided in the repository's main README, serving as the technical baseline required to understand the granular system specifications detailed across subsequent manuals.

---

## 🧩 Global System Architecture

The ecosystem isolates its automation domains across three functional, decoupled processing tiers:

### 1. Sensing & Human-Machine Interface (HMI) Tier – Thermostat Nodes
* **Compact Peripherals:** ESP8266 nodes equipped with a 1.8" SPI display layer.
* **Premium Touch Interfaces:** ESP32 nodes running FreeRTOS with a 3.5" TFT touchscreen display.
* **Sensor Matrix Array:** Continuous sampling of ambient temperature, relative humidity, CO₂, and TVOC indices.
* **Edge UI Functions:** Localized 24-hour historical trend charts, weekly scheduling adjustments, context-driven dynamic status icons, and 6-hour local weather forecast maps.

> 🚫 **Decoupled Control Rule:** Ambient thermostats do **NOT** directly actuate physical manifold electrovalves. Instead, they commit state calling configurations to the central persistence layer.

---

### 2. Physical Actuation Tier – Local Node Controllers
* **Manifold Hardware Edge:** ESP8266 (Lolin D1 Mini Lite) nodes managing multi-channel relay modules.
* **Electrical Switching:** 230V electrovalve manifold actuation driven by localized transistor-to-relay circuits.
* **Hydraulic Safeties:** Integrated software protection algorithms executing strict operational timeouts to damp fluid-hammer effects and protect plumbing loops.
* **Telemetry Auditing:** Updates execution confirmations back into the central database ledger upon finishing mechanical cycles.

This tier physically handles the distribution of hydronic thermal mass across the floor radiant infrastructure zones.

---

### 3. Central Engine Tier – Raspberry Pi 2B Server
* **Persistence Engine:** Local **MariaDB Database Server** coordinating the absolute state vector of the entire system network.
* **Core Automation Engine:** Python 3 background services processing *Extra Heating* cycles, predictive thermal inertia computations, and multi-node synchronization loops.
* **Local Web Interface:** Fully private local web server rendering comprehensive system diagnostic and real-time environment dashboards.
* **Data Science Services:** Automated cron routines query meteorological APIs and perform long-term big-data telemetry analytics.

This centralized layer serves as the unified "brain" that coordinates ambient room calls, structural manifold actuation, and hybrid thermal energy generation.

---

## 🔥 HVAC Orchestration Logic

Ambient thermostat terminals transmit two independent target configuration variables to the database:
* `WARM_STATE` — The current boolean target evaluation calculated against the weekly scheduled hourly profile.
* `WARM_FORCE` — A strict, event-driven manual user override flag (`Normal`, `Forced ON`, `Forced OFF`).

The centralized Python engine continually cross-references these specific variables alongside:
* Photovoltaic micro-generation capabilities (driving the solar *Extra Heating* optimizer).
* Live hybrid biomass wood thermo-stove water jacket temperature loops.
* Core energy plant ignition configurations (Auxiliary Boiler / Electric Heat Pump states).
* Boundary environment tracking records (Internal room telemetry vs. Outdoor weather vectors).

---

## 🌞 Solar Extra Heating Mode

Whenever background loops validate a substantial net solar production surplus:
* The central engine updates the network parameters, forcing room terminals into the automated solar optimization state (**Yellow Flame** icon on thermostats).
* The thermostat firmware enforces localized environmental safety margins, automatically dropping out individual zones if the room temperature breaches a strict **22°C ceiling**.
* Effectively turns the radiant concrete floor mass into a structural thermal buffer, heavily reducing reliance on grid power.

---

## 🔥 Hybrid Biomass Thermo-Stove Integration

When the independent biomass monitoring controller validates that the wood thermo-stove water jacket has crossed its core threshold (60°C):
* It assumes absolute thermodynamic priority, serving as the dominant heat source for the whole property.
* Pushes status mutations to the storage layer, updating local thermostat UIs to display the dedicated **STUFA (Stove)** indicator.
* The centralized plant routing script suppresses electric heat pump compressor activation, while distribution routines stagger hydronic flow sequences to prevent structural thermal shocks and loop oscillations.

---

## 📡 Edge-Node Network Communication Stack

Every distributed peripheral module interacts with the central Raspberry Pi server exclusively through:
* A localized, non-exposed, secure home **Wi-Fi infrastructure**.
* Flat, targeted SQL queries executed directly against dedicated table layouts.
* Automated server-side database **triggers** implementing event-driven updates.
* Single-row `_now` high-speed **cache tables** engineered to prevent edge microcontroller polling latency.

> 🔒 **Cloud Sovereignty:** The entire system architecture functions **100% locally**, utilizing zero external cloud dependencies or third-party web endpoints.

---

## 📑 Core Documentation Index

The following specific technical guides detail the structural components introduced in this manual:
* **MCU Hardware Reference:** Input/output layouts, components lists, and transistor wiring diagrams.
* **Database Schema Reference:** Complete table structures, column definitions, and SQL triggers.
* **Algorithmic Decision Manual:** Comprehensive tracking of the *Extra Heating*, biomass interlocks, and predictive logic cycles.
* **Data Flow Diagrams:** In-depth monospaced sequence trees mapping system transitions.
* **Firmware Documentation:** Structuring embedded firmware code bases and FreeRTOS task splits.
