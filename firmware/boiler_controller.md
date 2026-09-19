# Central Boiler & Recirculation Pump Controller – ESP8266 Firmware

🇮🇹 *Per la versione in italiano, clicca [qui](boiler_controller.it.md).*

The boiler controller manages the operational state of the centralized manifold recirculation pump and the primary electrical heating plant (Heat Pump/Auxiliary Boiler) based on aggregated zone loads and hydraulic system criteria.

As one of the most critical execution nodes in the Home Heating IoT ecosystem, this firmware guarantees:
*   ✅ Uninterrupted thermal delivery and automated grid load distribution.
*   ✅ Centralized heat exchanger protection routines.
*   ✅ Deterministic synchronization with hybrid biomass stove loops and server optimization algorithms.
*   ✅ Strictly controlled ignition and shutdown dampening sequences (short-cycling prevention).
*   ✅ Resilient standalone execution loops engineered to self-heal during transient Wi-Fi drops.

---

## 1. System Integration Architecture

The peripheral controller runs an independent, decoupled lifecycle routine:
*   Establishes local secure Wi-Fi infrastructure authentication.
*   Spawns a direct TCP transaction targeting the core database backend utilizing the native `MySQL_MariaDB_Generic` driver framework.
*   Queries the live aggregated requirement grid computed within the `zone_status` lookup cache.
*   Monitors energy parameters, biomass events, and optimization flags recorded in the `heating_state` ledger.
*   Orchestrates physical hardware line configurations:
    *   🔹 High-volume manifold **recirculation pump** relay.
    *   🔹 Primary electrical **heating plant burner / compressor** interlocks.
*   Applies strict mechanical delay windows to damp thermal shock and mechanical stress.
*   Commits execution metrics back into the `heating_state` time-series diagnostics database.

> 🧠 **Stateless Framework Principle:** The edge controller layer contains no hardcoded structural state dependencies. The absolute mathematical truth driving the HVAC state engine is mediated entirely through the central database layer.

---

## 2. Hardware Specification Baseline
*   **Edge MCU Core:** ESP8266 NodeMCU 12-E 1.0 Module layout.
*   **Switching Interface:** Isolated industrial electro-mechanical relays or solid-state relays (SSR) driving plant lines.
*   **Power Isolation Network:** Filtered DC-DC buck step-down configuration coupled with linear hardware low-dropout (LDO) regulators to shield the MCU core from high-voltage relay noise.
*   **Thermal Aggregation Input:** Dedicated digital interface routing lines, allowing integration of physical water loop immersion thermometers (optional/auxiliary telemetry tracking).

---

## 3. Direct MySQL/MariaDB Core Driver Layer

For all embedded assets built on top of the Espressif ESP8266 architecture, network transaction queries execute directly from the C++ firmware using the native `MySQL_MariaDB_Generic` library asset.

### Security & Privilege Profiles:
*   Authenticates via a **highly restricted database account profile** isolated to specific runtime queries.
*   Grants `SELECT` permissions targeting the immediate lookup cache elements.
*   Grants `INSERT` permissions to append time-series historical running records.
*   Zero administrative database privileges (`CREATE`, `ALTER`, `DROP`) are exposed to the edge network interface.

---

## 4. Algorithmic Processing Loop

The firmware operates on a structured, repetitive cron-like evaluation frame running exactly every 90 seconds:

```text
MAIN FIRMWARE OPERATIONAL LOOP (Every 90 Seconds)
 ├──> Execute SQL SELECT against zone_status table
 ├──> Evaluate if at least one hydronic loop reports an active state (1)
 ├──> Compute the optimized physical plant response matrix:
 │     ├──> De-energize whole system (All Idle)
 │     ├──> Energize distribution pump exclusively (Circulation Only)
 │     └──> Energize pump followed by managed burner firing (Full Heat Call)
 ├──> Enforce mechanical protective hardware delays
 ├──> Toggle corresponding electrical relay output lines
 └──> Push transaction confirmation log row to heating_state
```

---

## 5. Production SQL Query Specifications

### 🔍 5.1 Querying Live Actuator Status Metrics (Table: `zone_status`)
```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo
FROM zone_status
WHERE location='ceresole';
```

### 🔍 5.2 Appending Chronological Plant Ledger Logs (Table: `heating_state`)
```sql
INSERT INTO temperature.heating_state
(distributor_zone, heating_status, reason, pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo, termostufa)
VALUES ('%s', %d, 'HEATING OFF DUE TO TERMOSTATS OR STOVE REQUEST', %d, %d, %d, %d, %d, %d, %d, %d);
```
*(The parameterization script injects the target identifier string, global state bits, localized environment metrics, and the current binary biomass activity bit).*

---

## 6. Staged HVAC Activation Logic & Scenarios

### 6.1 Scenario A – Standard Zone Demand (At least one zone reporting active)
*   The controller instantly pulls the high-volume manifold distribution pump relay to high (**Pump = ON**).
*   Executes a protective **30-second fluid stabilization delay** to allow the mechanical electrovalves to finish their stroke configurations.
*   Drops in the electrical heat pump burner command lines (**Boiler = ON**).

### 6.2 Scenario B – Biomass Hybrid Integration (Thermo-Stove Active)
*   The wood stove node commands global thermostat zone adjustments. 
*   The boiler controller instantly matches the demand profile: activates the recirculation distribution pump (**Pump = ON**) and follows with the standard **30-second pre-circulation protective timeout loop**.
*   **Automatic Thermal Suppression:** If the high-temperature water entering from the biomass exchanger pushes the internal boiler storage tank parameter past the hardware setpoint (**49°C** in this implementation), the electric heat pump automatically shuts down its compressor banks, relying entirely on wood energy.

### 6.3 Scenario C – Photovoltaic Surplus Optimization (Extra Heating Mode)
*   The background optimization engine forces all room scheduling maps into call state.
*   The boiler controller activates the distribution pump (**Pump = ON**) and fires the system following the **30-second delay windows**.
*   **Comfort Ceilings Safety:** Individual thermostat terminals maintain local veto blocks. If a room handles solar gains and crosses the **22°C threshold**, it drops its zone row back to zero. If all rooms hit 22°C, the boiler engine instantly identifies the empty matrix and cuts power to prevent energy waste.

### 6.4 Scenario D – System Load Clear (Zero active loops remaining)
*   Instantly cuts power to the electric heat pump / burner relays (**Boiler = OFF**).
*   Maintains distribution pump operation for a **30-second post-circulation delay** to flush residual high-temperature thermal mass away from internal plant components.
*   De-energizes the pump line (**Pump = OFF**).

---

## 7. Fault Tolerance & Hardened Interlocks

### 7.1 Local Wi-Fi Connection Loss Self-Healing
*   Maintains the absolute current relay states to prevent sudden temperature drops.
*   Initiates background network scanning routines.
*   If network isolation passes a strict **30-second hardware boundary**: executes a hard MCU watchdog reboot, clearing relay outputs to default open circuits (forcing components **OFF** until database handshake validation recovers).

### 7.2 Database Incoherence Protection
*   If the database connection returns empty arrays or invalid formatting vectors: the firmware executes a safety lockout routine.
*   Instantly drops all plant outputs to zero, records a high-priority hardware fault block to the serial diagnostic interface, and pushes a critical exception notice to the error log ledger.

---

## 8. Logging, Diagnostics & Data Analytics
The embedded firmware continuously streams operation metrics across the hardware UART interface alongside the SQL tracking engines, featuring granular verbose log configurations:
*   Real-time compressor and electrical burner operational milestones.
*   Distribution loop configuration grids and calculated return delays.
*   Biomass injection parameters and thermal sensor validations.
*   Direct MariaDB transaction status alerts and query parsing metrics.
*   Local micro-second clock synchronization timestamps.

The comprehensive database footprint stored within `heating_state` provides the analytical background datasets needed to run season-over-season performance audits.

---

## 9. Future Implementation Roadmap
*   Direct integration with digital hydronic volumetric flow meters.
*   Advanced predictive logic tuning to minimize compressor activation frequencies.
*   Automated anti-legionella cycling scripts for domestic hot water (DHW) loops.
*   Self-learning activation delay calculations tracking real-world envelope responses.
*   Localized private web portal serving edge maintenance configurations.
