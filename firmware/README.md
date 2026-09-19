
# Distributed Firmware Architecture & Embedded Systems Code

This directory houses the embedded C++ source code, configuration files, and firmware components driving the distributed microcontroller nodes (ESP32 and ESP8266) across the Home Heating IoT ecosystem.

The software layer is designed around an event-driven, decoupled architecture. Peripheral nodes do not communicate directly with one another; instead, they interact asynchronously by executing flat SQL operations against the central MariaDB caching tables (`_now` views).

---

## 🧩 Firmware Tier & OS Structures

The firmware repository is organized into three distinct execution styles tailored to specific hardware capabilities:

### 1. ESP32 Premium Touch Nodes – Real-Time OS (FreeRTOS)
The firmware for the main living area thermostats leverages the dual-core multitasking capabilities of **FreeRTOS**. To ensure a lag-free graphical user interface (HMI) while executing blocking network sockets, operations are divided into prioritized task loops:
*   `Task_Sensors` (High Priority) — Continuous high-frequency querying of ambient metrics (BME280/MQ-135) via I2C/GPIO.
*   `Task_Logic` (Medium Priority) — Computes thermal hysteresis, user overrides (`WARM_FORCE`), and safety ceiling limits.
*   `Task_Display` (Medium Priority) — Handles TFT screen buffer buffer drawing and touchscreen interaction processing.
*   `Task_Network` (Low Priority) — Asynchronously handles local Wi-Fi re-connections and SQL transactions against MariaDB.

### 2. ESP8266 Standard Nodes – Sequential State Machines
The firmware for secondary bedrooms, bathroom thermostats, and weather stations is built on top of the classic Arduino framework using cooperative non-blocking timing structures (`millis()`-based state machines):
*   **Ambient Thermostats (ESP12-E):** Executes lightweight sequential cycles to read local temperature sensors, evaluate schedule maps, and handle physical button interrupts.
*   **Outdoor Weather Station (Wemos D1 Mini Pro):** Wakes up to sample barometric, temperature, and humidity sensors every 5 seconds, computes a 10-minute rolling average, and pushes a clean time-series dataset to the database server.

### 3. ESP8266 Manifold & Plant Controllers – Event-Driven Polling
The firmware driving the physical actuation loops is optimized for hardware safety and reliability:
*   **Manifold Valve Controllers (Lolin D1 Mini Lite):** Runs a continuous 60-second polling routine that queries the `termostat_warming_request` cache row. It implements a soft-start **90-second mechanical lag adjustment** to protect the physical electrovalves from quick pressure shifts.
*   **Central Boiler Plant Node (ESP12-E):** Evaluates aggregated feedback matrices and strictly controls burner ignition delays and protective **90-second pump pre-circulation loops**.

---

## 🔒 Edge Hardening & Network Resilience

Every embedded firmware module implements strict self-healing routines to guarantee high availability within the home:
*   **Hardware Watchdog Timers (WDT):** Enabled across all nodes to automatically force a hard hardware reset if a firmware loop experiences a freeze or a race condition.
*   **Asynchronous Wi-Fi Reconnect:** Network dropouts do not block local sensing loops. If the connection to the local router drops, the `Task_Network` thread or the `millis()` loop gracefully attempts re-connection in the background while local thermal safeties remain active.
*   **Database Transaction Protection:** If the central MariaDB server becomes unreachable, firmware elements default to a conservative standby layout, protecting active systems from executing unvalidated actions.

---

## 📁 Directory Directory Organization

As development progresses, this directory will be populated with production-ready codebases:
*   `📂 esp32_touch_thermostat/` — Complete platformio/arduino project for the premium 3.5" touchscreen nodes.
*   `📂 esp8266_button_thermostat/` — Lightweight firmware code for the button-operated 1.8" SPI display modules.
*   `📂 esp8266_manifold_controller/` — Relay driver firmware with mechanical timeout safety blocks.
*   `📂 esp8266_boiler_controller/` — Control code executing pre-circulation routines and interlocks.
*   `📂 esp8266_outdoor_weather/` — Low-power data acquisition code for the atmospheric sensor package.
