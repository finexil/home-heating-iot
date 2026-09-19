# Comprehensive System Documentation & Technical References

This directory serves as the centralized repository for deep-dive technical manuals, thermodynamic analysis reports, and systemic operating guidelines for the Home Heating IoT ecosystem.

While the primary repository directories (`database/`, `algorithms/`, `firmware/`, `hardware/`) store production-ready code, schemas, and blueprints, this section contextualizes how those components interact within a real-world residential infrastructure.

---

## 📌 Documentation Matrix

To navigate the specific operational layers of the system, reference the following upcoming documentation modules:

### 1. System Integration & Device Protocols
* **`devices.md`**: Outlines the network communication stack. Details how distributed ESP32 and ESP8266 microcontroller nodes manage asynchronous WiFi connections and interface with the MariaDB high-speed caching layers (`_now` tables).

### 2. HVAC Infrastructure & Hydraulic Engineering
* **`hydronic_circuits.md`**: Documents the physical plumbing topology, detailing the integration between the primary electric heat pump, the auxiliary boiler relays, and the hybrid biomass thermo-stove water jacket loop (operating on the 55°C–60°C hysteresis).

### 3. Thermal Envelope & Zone Mapping
* **`building_envelope.md`**: Provides structural telemetry analysis derived from over 24 months of historical time-series logging. Maps the specific thermal mass degradation and radiant slab inertia behaviors of the 8 managed heating zones distributed across the multi-floor property.

### 4. End-User Manual & Operational Guidelines
* **`user_manual.md`**: A clean reference manual defining the human-machine interface (HMI) logic, explaining thermostat touchscreen dashboard feedback parameters, manual overrides (`WARM_FORCE`), and air quality exchange safety overrides (CO₂/TVOC monitoring).

---

## 🏗️ Core Engineering Design Goals Documented Here

The architectural literature compiled in this directory focuses on four strict domestic automation goals:
* **High Availability Edge Independence:** Keeping the home fully automated, secure, and deterministically functional during complete external cloud or internet connection dropouts.
* **Energy Consumption Optimization:** Shifting thermal load demands directly into daytime solar micro-generation windows (**Extra Heating** optimization via SolarEdge API tracking).
* **Mechanical Shock Prevention:** Hardware protection mapping that mitigates relay fatigue, plumbing cavitation, and short-cycling wear on expensive central compressors.
* **Granular Air Safety Interlocking:** Continuous sensor aggregation designed to handle automated ventilation cycles when operating solid-fuel hybrid biomass heat sources indoors.
