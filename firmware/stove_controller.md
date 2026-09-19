# Biomass Thermo-Stove Controller – ESP8266/ESP32 Firmware

🇮🇹 *Per la versione in italiano, clicca [qui](stove_controller.it.md).*

The thermo-stove controller monitors the operational status of the hybrid biomass stove and measures the fluid loop temperature feeding the primary boiler heat exchanger.

The core responsibilities of this embedded node are:
*   ✅ Informing the central automation layer when the biomass thermo-stove transitions to active.
*   ✅ Synchronizing system states with centralized plant routines (boilers, pumps, and valves).
*   ✅ Pushing structural schedule adjustments to force climate zones into "Stove Heating" mode.
*   ✅ Ensuring thermal envelope safety and maintaining deterministic runtime loops.

Although structurally straightforward, this controller is a critical component that establishes thermodynamic priority across the whole property.

---

## 1. System Integration Architecture

The peripheral controller operates as a decoupled sensory station:
*   Queries a physical **water loop immersion temperature sensor** (thermal switch layout mounted directly on the boiler storage tank wrapper).
*   Determines if the biomass thermo-stove is generating active, usable thermal energy.
*   Spawns a direct TCP transaction targeting the core database backend utilizing the native `MySQL_MariaDB_Generic` driver framework.
*   Pushes dynamic `stove_on` status mutations to the `termostufa` log ledger and the `zone_status` lookup cache.
*   Asserts the priority flag `f1 = 1` inside the `TermostatSetup` configuration layout. This lets distributed thermostats, manifold actuators, and the central boiler identify "STOVE" mode.

> 🚫 **Decoupled Architecture Rule:** The edge microcontroller **NEVER** directly operates physical manifold zone valves or fires the primary plant infrastructure. It exclusively serves as a localized energy availability status provider.

---

## 2. Hardware Specification Baseline
*   **Edge MCU Core:** ESP32 / ESP8266 D1 Mini series form factor.
*   **Thermal Sensing Interface:** High-temperature PT100/NTC thermistor coupled with an analog-to-digital converter (ADC) module.
*   **Digital Interlock Input:** Isolated optocoupler or dry contact input routing lines tracking the thermo-stove mechanical thermal switch status.
*   **Physical Switching Output:** Onboard transistor-driven relay output channel orchestrating the high-temperature local recirculation pump feeding the boiler.
*   **Power Isolation Network:** Filtered and stabilized 3.3Vdc bus layout shielded from high-voltage inductive pump noise.

---

## 3. Thermal State Evaluation & Hysteresis Logic

### 🌡️ Operational Plant Thresholds
*   **Biomass Stove Active (ON State):** Initiated as soon as the boiler tank temperature parameter breaks the **60°C threshold** alongside a verified closing transition of the physical thermal switch.
*   **Biomass Stove Standby (OFF State):** Initiated when the fluid loop curve falls below the **55°C boundary** alongside an open transition of the physical switch.

> ⏱️ **Mechanical Stabilization Strategy:** During transient heating or burn-down phases, the system experiences short-term fluid temperature fluctuations as cold water enters from the radiators. This can cause the thermal switch to oscillate. To prevent rapid relay clicking and protect the recirculation pump from excessive wear, the firmware injects specialized **software timeout dampening loops**. When a state shift occurs, the pump is locked in its current position until the timeout window expires.

### 📊 Thermal Target Mapping

```text
+───────────────────────+─────────────────+─────────────────────────────────────────+

| Measured Temperature  | Logical Status  | Recirculation Pump Mechanical State     |
+───────────────────────+─────────────────+─────────────────────────────────────────+

| Temp > 60°C           | stove_on = 1    | Pump = ON (Forced active flow)          |
| Temp < 55°C           | stove_on = 0    | Pump = OFF (Idle standby)              |
| 55°C <= Temp <= 60°C  | Retain Previous | Maintains current state until timeout   |
+───────────────────────+─────────────────+─────────────────────────────────────────+
```

---

## 4. Native Direct MySQL Caching Layer

For all embedded assets built on top of the Espressif MCU architecture, network transaction queries execute directly from the C++ firmware using the native `MySQL_MariaDB_Generic` library asset.

### 🔒 User Account Privilege Profiles:
*   `SELECT` (To evaluate current zone co-dependencies).
*   `INSERT` (To append time-series historical running records inside the `termostufa` log).
*   `UPDATE` (To modify the real-time cache row parameters).

---

## 5. Production SQL Query Specifications

### 🔍 5.1 Scanning Persistent System Matrix (Table: `zone_status`)
```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo 
FROM temperature.zone_status 
WHERE location='ceresole';
```

### 🔍 5.2 Appending Chronological Stove Ledger Logs (Table: `termostufa`)
```sql
INSERT INTO temperature.termostufa (Stato, Condizione, Temperatura)
VALUES ('## stato termostufa ##', 0/1, 0/1);
```

### 🔍 5.3 Staggered Zone Activation Sequence (Table: `TermostatSetup`)
To distribute biomass energy without dropping the water loop temperature too quickly, the firmware implements a **staggered room group timing algorithm**. It sets the priority flag `f1 = 1` sequentially across the zones. When the stove burns down, the flag `f1 = 0` is cleared simultaneously across all rooms to restore normal operations instantly.

**Group 1 Ignition – Bathroom Zones:**
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%bagno';
```

**Group 2 Ignition – Bedroom Zones:**
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%camera';
```

**Group 3 Ignition – First Floor Living Room:**
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room = 'p1 soggiorno';
```

**Group 4 Ignition – Ground Floor Living Room:**
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room = 'pt soggiorno';
```

**Group 5 Ignition – Terrace / Ex-Barn Studio Loop:**
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%terrazzo';
```

---

## 6. Firmware Flowchart

The main execution loop automatically scales its background polling interval based on active heating states:

```text
MAIN FIRMWARE LOGIC LOOP
 ├──> Sample analogue immersion sensor and read digital thermal line state
 ├──> Evaluate current temperature matrix against hysteresis boundaries
 ├──> IF (State Shift Confirmed & Safety Timeout Window Expired):
 │     ├──> True status active (stove_on = 1):
 │     │     ├──> Scale main loop delay frequency to 40 MILLISECONDS
 │     │     ├──> Execute sequential staggered SQL UPDATE statements (f1 = 1)
 │     │     └──> Push live event record to termostufa database row
 │     └──> True status idle (stove_on = 0):
 │           ├──> Scale main loop delay frequency to 6 SECONDS
 │           ├──> Execute global simultaneous SQL UPDATE statements (f1 = 0)
 │           └──> Push standby event record to termostufa database row
 └──> Complete system telemetry diagnostic check
```

---

## 7. Integrated Subsystem Interaction

The central automation engine and peripheral thermostat nodes monitor the state mutations published by the stove node, executing standardized override rules:

### 💡 When `stove_on = 1` is Asserted:
*   Distributed thermostat terminals capture the `f1` flag modification, shift their visual screen buffers to display the dedicated **STUFA (Stove)** logo, and instantly update `termostat_warming_request`.
*   Standard weekly scheduling rules and active solar *Extra Heating* optimization cycles are bypassed.
*   The primary electric heat pump compressor banks are completely locked out to ensure zero grid running costs.
*   The high-temperature local recirculation loop pump is driven based on active zone demand parameters.

### 💤 When `stove_on = 0` occurs:
*   System variables gracefully return control back to normal daily schedule routines.
*   The automated solar self-consumption algorithm is re-enabled.
*   The electric heat pump/compressor plant returns to standby availability, ready to fire if needed.

---

## 8. Fault Tolerance & Self-Healing Fallbacks

### 🛡️ Debounce-Hardened Hysteresis
Blocks erratic switching behaviors when fluid boundaries hover precisely between 55°C and 60°C.

### 🛡️ Immersion Sensor Timeout Protections
If the analog thermistor array stops responding or returns open circuit anomalies: the firmware freezes the last known valid parameters, pushes a critical warning to the serial interface, and initiates a safe fallback shutdown sequence after a predefined cycle count.

### 🛡️ Wi-Fi Disruption Recovery Loop
*   Maintains localized relay configurations to prevent abrupt temperature changes.
*   Queries connection matrices every 5 seconds.
*   If network isolation passes a strict threshold: executes a hard MCU watchdog reset, de-energizing the local recirculation pump relays and clearing the DB state as soon as host connection recovers.

---

## 9. Comprehensive System Diagnostic Ledgers

Every transition milestone is logged both across the hardware serial UART line and directly into the central database `heating_state` index, compiling a clean time-series record including precise transaction timestamps, binary biomass states, primary boiler configurations, and active manifold positions.
This persistent ledger provides the analytical background datasets needed to audit structural biomass contributions across changing winter seasons.

## 10. Future Implementation Roadmap
*   Integration of dual thermistor monitoring arrays tracking scambiatore entry and exit boundaries.
*   Dynamic thermodynamic efficiency curve mapping calculated directly over time.
*   Direct serial coupling with digital volumetric flow and thermal power consumption meters.
*   Flue gas temperature monitoring for accelerated stove state detection.
*   Localized private web interface serving real-time maintenance configurations.

