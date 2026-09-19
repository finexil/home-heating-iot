# Biomass Thermo-Stove Logic – Central Server Routine

🇮🇹 *Per la versione in italiano, clicca [qui](stove_logic.it.md).*

This document outlines the **biomass thermo-stove automation logic** implemented within the central Python engine running on the Raspberry Pi. This core module orchestrates the interactions between the primary electric heat pump, auxiliary boiler relays, hydronic manifold loops, and localized ambient thermostats.

Within the Home Heating IoT framework, the biomass thermo-stove is treated as the **absolute highest priority heat source**, overriding standard heat pump calls and active photovoltaic surplus optimization algorithms.

---

## 1. Architectural System Integration

Within this decentralized network layout, the thermo-stove subsystem operates under specific constraints:
*   ✅ Does **NOT** directly switch manifold zone electrovalves.
*   ✅ Does **NOT** directly trigger heat pump or boiler hardware components.
*   ✅ Provides a **critical system-wide energy status flag** to the persistence layer.
*   ✅ Is actively audited by a standalone dedicated edge microcontroller node.
*   ✅ Shifts the decision matrices computed by the centralized background processes.

The thermodynamic availability of the biomass source is propagated exclusively through localized MariaDB transaction states.

---

## 2. Telemetry Origin: "Active Thermo-Stove"

The standalone edge controller tracking the biomass subsystem executes the following runtime check:
1. Samples the physical water jacket temperature inside the storage exchanger tank.
2. Applies a strict mechanical safety hysteresis: **ON when loop data > 60°C**, **OFF when loop data < 55°C**.
3. Upon validation, commits the live state variable directly into the `stove` field of the `zone_status` ledger and asserts the `f1` flag inside `termostatSetup`.

> 💡 **Decoupled Validation:** The central server does not process raw analogue sensor data from the stove; it trusts the validated state updates committed to the database layer by the dedicated microcontroller node.

---

## 3. Relational Mapping Interfaces

The biomass routine tracks environmental variables across the following persistent state models:

| Target Table | Strategic Purpose |
| :--- | :--- |
| `termostatSetup` | Evaluates dynamic hourly room profiles and multi-source scheduling flags. |
| `heating_state` | Coordinates the active power states of boilers, stoves, and pumps. |
| `zone_status` | Audits the actual mechanical deployment position of physical electrovalves. |
| `termostat_warming_request` | Verifies real-time aggregated calling configurations from room terminals. |
| `termostat_temp_now` | References immediate room temperatures to enforce localized safety ceilings. |

---

## 4. Operational Heat Source Hierarchy

The system enforces a rigid thermal priority framework during every algorithmic decision cycle:

1. <img width="39" height="39" alt="image" src="https://github.com" /> **Biomass Thermo-Stove** (Zero operational grid cost - High priority)
2. <img width="39" height="39" alt="image" src="https://github.com" /> **Extra Heating** (Solar photovoltaic micro-generation surplus)
3. <img width="40" height="39" alt="image" src="https://github.com" /> **Primary Plant** (Electric Heat Pump / Boiler - Standard grid tariff)

---

## 5. Algorithmic State Machine Transitions

### 5.1 Active Biomass Source State (`stove_on = 1`)
When the dedicated controller validates that the thermo-stove loop has crossed the target temperature threshold:
*   ✅ The electric heat pump/compressor operation is **forced into a locked-out OFF state** by the internal regulation scripts to shield the building from drawing grid power.
*   ✅ Active solar **Extra Heating logic loops are completely bypassed** to avoid structural overheating.
*   ✅ The thermo-stove circulation loops run continuously; meanwhile, the primary manifold recirculation pump is dynamically driven based on individual room requirements.
*   ✅ Ambient room thermostat screens update their user interfaces to display the **STUFA (Stove)** indicator.
*   ✅ Hydronic zone loops receive warm water flow only if local environment telemetry tracks below the strict **22°C maximum safety threshold**.

> ⚠️ **Dynamic Control Rule:** The active biomass stove does **not** blindly flood all hydronic manifolds. Local ambient thermostats retain absolute veto control over whether an individual room accepts thermal energy.

### 5.2 Standby/Offline Biomass Source State (`stove_on = 0`)
When the biomass stove burns down and drops below the hysteresis cutoff point:
*   The automation engine returns system parameters back to standard daily profiling rules.
*   The **Extra Heating algorithm is re-enabled**, allowing background routines to monitor solar micro-generation metrics.
*   The primary plant infrastructure (Heat Pump) returns to standby availability, ready to respond to standard thermostat scheduling tables.
*   Manifold valve deployments return to tracking standard hourly setpoints defined within `termostatSetup`.

---

## 6. Thermostat Interface & Visual Feedback

When `stove_on = 1` occurs:
*   Peripheral thermostat hardware arrays **maintain their standard internal validation processes**.
*   Nodes continue to parse the active weekly scheduler matrix alongside the `f0` (Solar) and `f1` (Biomass) operational flags.
*   Display layers inject the custom **<img width="39" height="39" alt="image" src="https://github.com" /> STUFA** icon onto the screen buffer to explicitly show the true active heat source feeding the room loops.

---

## 7. Recirculation Pump Orchestration

The central plant hydronic manifold distribution pump operates under independent security constraints:
*   ✅ Operates smoothly to distribute thermal mass even when the primary heat pump compressor is locked out.
*   ✅ Shifts into an active state whenever at least one structural room zone layout declares an open position.
*   ✅ Respects the predefined **90-second safety dampening delay** handled by the controller logic.

During active biomass stove cycles, the distribution pump runs exclusively to balance heat across the floors, while the primary electrical compressors are held offline.

---

## 8. Solar Extra Heating Suppression

Whenever `stove_on = 1` is committed to the persistence layer, the automation script immediately sets `extra_heating_on = 0`, regardless of current inverter array metrics or battery bank capacity status.

### Engineering Motivations:
* The active wood stove already satisfies the building's thermal envelope criteria.
* Blocks concurrent activation conflicts between independent thermal sources.
* Prevents unnecessary electrical power draw from the auxiliary heat pump components.
* Protects central plant exchangers from experiencing simultaneous high-temperature injections.

The solar self-consumption optimization loop returns to service automatically as soon as the thermo-stove transitions back into a cold standby state.

---

## 9. System Hardening & Safe Interlocks

The background automation scripts and hardware edge controllers apply rigorous validation routines to defend mechanical plumbing loops:
*   **Edge-Driven Hysteresis:** The stove microcontroller locks states to prevent rapid runtime state bouncing between 55°C and 60°C.
*   **Actuator Loop Validation:** Cross-references true physical manifold openings through continuous `zone_status` table reviews.
*   **Cavitation Prevention:** Automatically dumps heat or kills circulation calls if parameters indicate pumps are running against sealed manifolds.
*   **System Integrity Fallback:** In the event of network dropouts, database corruption, or reading corrupted values, the central server defaults to a **conservative safety state**, forcing boiler components offline and pushing debug notices to the error logs.

---

## 10. Audit Logging & Structural Performance Mapping

Every transition milestone is committed to the `heating_state` ledger, compiling a long-term operational record:
* High-precision transaction timestamps.
* Real-time biomass stove activity flags.
* Simultaneous primary boiler configurations.
* Aggregated circulation pump metrics and decision-reasoning strings.

**Telemetry Asset Value:**
* 🛠️ Immediate diagnostic tracing and debugging.
* 📈 Longitudinal system performance auditing across changing seasons.
* 🧠 Data sets to refine future predictive automation scripts.

---

## 11. Core Directory Matrix References

* **`extra_heating.md`:** Comprehensive breakdown of the solar self-consumption surplus routing strategies.
* **`boiler_controller.md`:** Hardware orchestration rules driving primary heat pump operations.
* **`stove_controller.md`:** Embedded C++ firmware documentation for the dedicated biomass edge microcontroller.
* **`overview.md`:** Core summary defining the global automation ecosystem architecture.
