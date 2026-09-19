# Premium Touch Thermostat – ESP32 Advanced Firmware Reference

🇮🇹 *Per la versione in italiano, clicca [qui](thermostat_esp32.it.md).*

This document delivers a deep-dive technical blueprint of the embedded firmware running on the **ESP32** microcontroller coupled with a **3.5" TFT touchscreen display**. Serving as the most advanced edge station within the ecosystem, this terminal is deployed across the home's primary macro-zones (Living Rooms, Studio Loops).

The codebase leverages the multithreading capabilities of **FreeRTOS** to orchestrate environmental sensor arrays, high-density capacitive graphical user interfaces, multi-page layout environments, and direct asynchronous server transactions.

---

## 1. Architectural System Overview

The application architecture targets the following design criteria:
*   ✅ Highly fluid and responsive human-machine interface (HMI).
*   ✅ Real-time multi-variable environmental sensor telemetry acquisition.
*   ✅ Non-blocking, high-availability direct database connectivity.
*   ✅ Asynchronous processing of multi-point historical data trend charts.
*   ✅ Real-time execution of localized logic matrices and context-driven status icons.
*   ✅ Strict operational task segregation to eliminate thread execution bottlenecks.

By isolating independent components into dedicated **FreeRTOS task threads**, the system guarantees that heavy database network latency or router dropouts can never freeze screen drawing or touchscreen interrupt handling.

---

## 2. Hardware Specification Baseline
*   **Edge MCU Core:** ESP32-WROOM / ESP32-DevKit series operating at a stabilized **240 MHz clock**.
*   **Display Layer:** 3.5" SPI TFT display panel featuring resistive touch processing (driven by the XPT2046 matrix controller).
*   **Sensory Array:**
    *   **BME280:** Environmental sensor capturing localized barometric pressure, relative humidity, and temperature.
    *   **CCS811:** Indoor air quality sensor monitoring equivalent carbon dioxide (eCO₂) and total volatile organic compounds (TVOC).
*   **Power Circuit Isolation:** Centralized 5Vdc bus input regulated to 3.3Vdc by the MCU board logic. Includes a dedicated display backlight transistor controller running a hardware-protective **1-minute and 30-second automatic screen dimming timeout**.

---

## 3. Peripheral Pinout Specifications

### 3.1 Display Interface (ST7796 / ILI9488 Controllers)

| Functional Line | ESP32 GPIO Pin |
| :--- | :---: |
| **CS (Chip Select)** | 5 |
| **RESET** | 4 |
| **DC (Data/Command)** | 2 |
| **MOSI (SPI Data In)** | 23 |
| **MISO (SPI Data Out)** | N.C. (Not Connected) |
| **SCK (SPI Clock)** | 18 |
| **LED (Backlight Control)** | 16* |

*\*Pin 16 drives the gate of an external switching transistor circuit; it does not supply the display backlight directly.*

### 3.2 Touch Interface (XPT2046 Controller)

| Functional Line | ESP32 GPIO Pin |
| :--- | :---: |
| **T_CS (Touch Chip Select)** | 15 |
| **T_CLK (Touch SPI Clock)** | 18 |
| **T_DIN (SPI Data In)** | 23 |
| **T_DO (SPI Data Out)** | 19 |
| **T_IRQ (Touch Interrupt)** | N.C. (Not Connected) |

### 3.3 Core Sensors (I2C Bus Configuration)

| Hardware Sensor | Pin Allocation | Default I2C Address |
| :--- | :--- | :---: |
| **BME280** (Environment) | SDA = 21, SCL = 22 | 0x77 / 0x76 |
| **CCS811** (Air Quality) | SDA = 21, SCL = 22, WAC = GND | 0x5A |

---

## 4. FreeRTOS Task Architecture & Thread Priority Matrix

The firmware splits execution blocks into localized parallel task wrappers running deterministically:

### 4.1 `task_sensori` (High Priority)
*   Queries barometric, temperature, relative humidity, and air metrics at high frequency.
*   Applies running median filtering algorithms to stabilize input variance.
*   Updates shared internal variables using thread-safe memory boundaries.
*   **Execution Period:** 100 milliseconds.

### 4.2 `task_DB` (Medium Priority)
Manages the sequence of raw SQL transaction blocks targeting the server:
*   Processes dynamic `INSERT`, `UPDATE`, and `SELECT` query arrays.
*   SQL query strings are stored inside a localized thread-safe data matrix structure shared across active task loops.
*   **Execution Period:** 100 milliseconds.

### 4.3 `task_Time` (Medium Priority)
Coordinates time-series calendar metrics, network telemetry tracking, and localized automation clocks:
*   Triggers periodic cron-like routines (1-minute, 10-minute, and 24-hour schedules).
*   Manages Daylight Saving Time (DST) conversions automatically via NTP.
*   Tracks the 1'30" display backlight screen dimming countdown timer.
*   Orchestrates network-wide polling intervals to fetch secondary zone statuses and temperature vectors.
*   Monitors RSSI Wi-Fi signal power and triggers network resilience routines.
*   Fetches updated multi-hour weather predictions and room target configurations.

### 4.4 `task_Loop` (Default Priority Core Task)
Orchestrates primary application behaviors, touch event handling, and display redraw loops:
*   Computes thermostat thermal responses and manual override (`WARM_FORCE`) states.
*   Manages graphical interface redraw allocations, charting routines, and rendering operations.
*   **Execution Period:** 100 milliseconds.

---

## 5. Graphical User Interface (HMI Layouts)

### 📊 Main Climate Dashboard
*   Localized temperature, humidity, and barometric variables.
*   Granular air quality telemetry (rendered via a contextual dynamic CO₂/TVOC visual status bar).
*   Dynamic heating status graphics matching the active thermal source.
*   Unified system calendar time, network Wi-Fi connectivity flags, and live outside environmental data.
*   Active hourly target temperature threshold and 6-hour localized weather prediction loops.

### 📊 Historical 24-Hour Analytics Environment
*   High-precision historical coordinate charts logging internal temperature and humidity curves against outdoor weather metrics (compiled at 10-minute intervals).
*   Localized peak minimum and maximum mathematical indicators.
*   Weekly scheduling graph overlay mapped directly against the room's current temperature trend line.

### 📊 System-Wide Zone Infrastructure Map
*   Visual status matrix detailing the real-time temperature data and heating states of all distributed thermostats on the home network.
*   Visual color blocks: **Red** represents a verified active loop call; **Green** represents an idle zone state.

---

## 6. HVAC Orchestration Logic

The touch terminal dynamically updates its active visual icon layout to reflect the specific power source feeding the hydronic loops:

| Visual Icon | Structural System Meaning |
| :---: | :--- |
| <img width="40" height="39" alt="image" src="https://github.com" /> | **Standard Active Call:** Heat requested locally by the zone's hourly schedule rules. |
| <img width="39" height="39" alt="image" src="https://github.com" /> | **Extra Heating Mode:** System override driven by real-time solar photovoltaic net surplus. |
| <img width="39" height="39" alt="image" src="https://github.com" /> | **Biomass Stove Mode:** Hot water loops supplied entirely by the active wood thermo-stove. |

### Operational Optimization Guidelines:
*   Both *Extra Heating* and *Biomass Stove* modes command global loop activation.
*   To prevent structural overheating, the local firmware automatically blocks delivery if ambient room metrics cross the strict **22°C ceiling**.
*   The display layer clearly isolates and registers these states without cross-source conflicts.

---

## 7. Production Database Connection Specifications

The ESP32 platform authenticates and executes transactional relational operations utilizing the specialized direct `ESP32_MySQL` library driver.

### 🔍 7.1 Pushing Environmental Telemetry (Table: `powerDetails`)
```sql
INSERT INTO temperature.powerDetails (room, Temperature, Humidity, Pressure) 
VALUES ('%s', %.3f, %.3f, %.3f);
```
*(Parameters parse the explicit `room_id` string alongside raw float figures representing temperature, humidity, and pressure).*

### 🔍 7.2 Appending Chronological Room Calls (Table: `Warming_state`)
```sql
INSERT INTO temperature.Warming_state (room, FLG_ON, FLG_FORCE) 
VALUES ('%s', %d, %d);
```

### 🔍 7.3 Fetching Zone Profiles & Optimization Flags (Table: `TermostatSetup`)
```sql
SELECT * FROM temperature.TermostatSetup 
WHERE room='%s';
```
*Queries the primary scheduling matrices along with the solar optimization flag (`f0`) and the biomass priority flag (`f1`).*

### 🔍 7.4 Network Status Mapping Caching (Tables: `termostat_temp_now` & `zone_status`)
```sql
SELECT `pt soggiorno`, `pt camera`, `pt bagno`, `p1 soggiorno`, `p1 camera`, `p1 bagno`, `te terrazzo` 
FROM temperature.termostat_temp_now 
WHERE location='%s';
```
```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo 
FROM temperature.zone_status 
WHERE location='%s';
```

### 🔍 7.5 Parsing Meteorological Payload Ingestion (Table: `forecasts`)
```sql
SELECT temperature, wind_speed, icon, description 
FROM weather.forecasts 
WHERE timestamp = (SELECT MIN(timestamp) FROM weather.forecasts WHERE timestamp > CURRENT_TIMESTAMP());
```

---

## 8. Fault Tolerance & Hardened Self-Healing

The ESP32 application stack incorporates the following hardware protection layers:
*   ✅ Independent software watchdog counters auditing thread state changes.
*   ✅ Native hardware ESP32 Watchdog Timers (WDT) preventing CPU core lockups.
*   ✅ Continuous data consistency validation routines mapping sensor operational boundaries.
*   ✅ Automated emergency hardware reboots if network task threads stall indefinitely.
*   ✅ Hardened Wi-Fi socket drop retry timeout thresholds.
*   ✅ Resilient startup memory restoration loops executing after unexpected utility power blackout events.

## 9. Diagnostic Logging & Debug Specifications
The terminal streams runtime analytical logs across the native serial UART bus, featuring selectable verbosity tiers:
*   ✅ Real-time analog-to-digital sensor conversion maps and validation parameters.
*   ✅ Local Wi-Fi authentication status changes and network link metrics.
*   ✅ Display redraw intervals, coordinate matrix updates, and touchscreen calibration data.
*   ✅ Raw incoming relational configurations and query execution strings.
*   ✅ Database exception handling messages and sensor failure fallback triggers.
*   ✅ NTP-synchronized local server micro-second timeline checks.

##  10. Future Implementation Roadmap
*   ✅ Support for automated Over-The-Air (OTA) firmware flashing paths over secure Wi-Fi connections.
*   ✅ Optional MQTT application stack implementation for advanced integration profiles.
*   ✅ Native, direct integration out-of-the-box with Home Assistant ecosystem endpoints.
*   ✅ Dynamic mathematical smoothing logic to process volatile indoor air quality measurements.
*   ✅ User-selectable dashboard visual themes (Dark Mode / Light Mode interface layouts).
*   ✅ Localized on-screen settings layout panel for network and credential tuning.
*   ✅ Self-learning thermal logic mapping domestic usage patterns automatically over time.
