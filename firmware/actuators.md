# Manifold Electrovalve Actuator Controller – ESP8266 Firmware

🇮🇹 *Per la versione in italiano, clicca [qui](manifold_controller.it.md).*

This document describes the embedded firmware driving the physical actuation controllers based on the **ESP8266 Lolin D1 Mini Lite** platform. These edge units orchestrate the physical hydraulic electrovalves of the multi-zone hydronic radiant floor heating network. Each distributed controller board manages between 2 to 4 independent heating zones, depending on its specific manifold node assignment.

The manifold controllers represent the **physically active layer of the automated infrastructure**: they cycle the high-voltage manifold relays dynamically based on call configurations computed by remote thermostat terminals.

---

## 1. Algorithmic System Architecture

Each peripheral manifold controller runs an autonomous execution loop:
*   ✅ Establishes local non-blocking Wi-Fi authentication.
*   ✅ Spawns a direct TCP/IP **MySQL/MariaDB connection** targeting the central Raspberry Pi server.
*   ✅ Executes raw SQL query transactions to pull real-time zone requirements.
*   ✅ Directs onboard relay channels driving the 230V slow-actuating manifold electrovalves.
*   ✅ Updates verified physical execution parameters back into the `zone_status` schema.
*   ✅ Applies software protective constraints and hardware-protective timeout loops.
*   ✅ Self-heals local runtime failures during transient network dropouts.

> 🧠 **Stateless Firmware Principle:** Actuator edge units maintain no persistent logic states locally. The true intended system target always resides inside the central database layer.

---

## 2. Hardware Specification Baseline
*   **Edge MCU core:** ESP8266 Lolin D1 Mini Lite.
*   **Switching Array:** 230V or 12V heavy-duty mechanical relays driven via localized transistor networks and protective flyback diodes.
*   **Power Distribution Network:** Centralized 12Vdc bus input matched with an onboard DC-DC buck converter to step down power to +5Vdc, subsequently scaled to 3.3Vdc by the ESP board regulator.
*   **Visual Debug Indicators:** Optional diagnostic LED layout showing relay logic.
*   **Topology Mirroring:** Structurally identical circuit footprints are deployed across the Ground Floor and First Floor manifolds.

---

## 3. Native Direct MySQL Database Persistence

To minimize overhead and ensure robust connectivity across the embedded platform, the ESP8266 microcontrollers connect directly to the database backend utilizing the specialized `MySQL_MariaDB_Generic` library. This allows raw `INSERT`, `UPDATE`, and `SELECT` query strings to execute natively from the firmware without middleware overhead.

### Code Initialization Reference:
```cpp
MySQL_Connection conn((Client*)&wifi);
MySQL_Cursor cursor(&conn);
```

### Setup Diagnostics Sequence:
1. Enforces robust local Wi-Fi association parameters.
2. Initiates a direct TCP transaction targeting the MariaDB port on the server.
3. Authenticates using a **dedicated, restricted network user profile** limited strictly to the following privileges:
   *   `SELECT` (To read operational caches).
   *   `UPDATE` (To verify valve executions).
   *   `INSERT` (To append time-series diagnostics log rows).

> 🔒 The specialized user account possesses zero schema modification rights (`ALTER`, `DROP`) and is completely blocked from accessing tables outside its operational scope.

---

## 4. Firmware Flowchart

The main firmware runtime loop maps the execution sequence executed on a dedicated interval:

```text
MAIN FIRMWARE EXECUTION LOOP (Every 5–10 Seconds)
 ├──> Execute SQL SELECT against termostat_warming_request table
 ├──> Parse incoming array to identify zones assigned to this node
 ├──> Compute the required binary target state (ON / OFF)
 ├──> Evaluate mechanical safety dampening timeouts
 ├──> Toggle physical electrical relay channels
 ├──> Push SQL UPDATE transaction to mirror state into zone_status
 └──> Refresh local telemetry check timestamps
```

---

## 5. SQL Query Reference

### 🔍 5.1 Querying Target Configurations (Table: `termostat_warming_request`)
```sql
SELECT pt_soggiorno, pt_camera, pt_bagno
FROM termostat_warming_request
WHERE location='ceresole';
```
*(The targeted column arrays automatically shift to match the bedroom/bathroom zones on the First Floor manifold controller).*

### 🔍 5.2 Scanning Persistent System Matrix (Table: `zone_status`)
```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo
FROM temperature.zone_status 
WHERE location='ceresole';
```

### 🔍 5.3 Pushing Valve Execution Updates

**Hydronic Loop Deactivation (OFF State):**
```sql
UPDATE temperature.zone_status
SET pt_soggiorno=0, pt_camera=0, pt_bagno=0 
WHERE location='ceresole';
```

**Hydronic Loop Activation (ON State):**
```sql
UPDATE temperature.zone_status
SET pt_soggiorno=1, pt_camera=1, pt_bagno=1 
WHERE location='ceresole';
```

### 🔍 5.4 Appending System Time-Series Diagnostics Logs (Table: `heating_state`)
This logging routine appends structured time-series rows to allow retrospective diagnostics. It is purely informative and is never parsed by hardware edge assets to compute operational targets.

```sql
INSERT INTO temperature.heating_state
(distributor_zone, heating_status, reason, pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo)
VALUES ('%s', %d, '##stato', %d, %d, %d, %d, %d, %d, %d);
```
*(The parameters inject strings defining the target `zone_id`, binary `heating_status`, and the full boolean array representing all active loops).*

---

## 6. Mechanical Timeout & Latency Balancing

Physical electrothermal manifold actuators exhibit significant mechanical latency parameters:
*   Minimum physical opening envelope duration: ~90 seconds.
*   Minimum physical closing envelope duration: ~90 seconds.
*   Total completion timeframe for complete stroke mechanical deployment: up to **3 minutes (180 seconds)**.

To prevent hydraulic issues and avoid running the primary circulation pumps against completely sealed manifolds (cavitation hazard), a zone is logicialized as active or inactive as soon as the actuator crosses the **50% deployment mark (90 seconds into the mechanical cycle)**.

The embedded firmware implements:
*   ✅ Standalone software timing counters running independently for each zone channel.
*   ✅ Hardened configurable safe delay thresholds.
*   ✅ Transient lockout windows to damp rapid ON/OFF oscillation anomalies.
*   ✅ Proactive fallback protection layers if target metrics fluctuate too quickly.

---

## 7. Fault Tolerance & Self-Healing Fallbacks

The actuator node executes automated safe-state operations under the following network drop failure scenarios:

### 📡 Local Wi-Fi Connection Dropouts
* Re-evaluates connection matrices and retries router authentication loops every 5 seconds.
* If network loss persists past a strict **30-second boundary**, the MCU triggers a hard hardware reboot.
* During the reboot sequence, physical relays fall back to an open circuit layout, forcing all electrovalves into a safe **OFF** state while updating the DB state as soon as host connection recovers to prevent unmetered energy shifts.

### 🗄️ MariaDB / MySQL Server Inaccessibility
* Registers error logs via the hardware serial output bus.
* Sequentially executes retry connection strings.
* Freezes current physical valve deployment configurations, executing no relay modifications until transactional validation returns.

### ❌ SQL Packet Read Failures
* Defaults immediately to the absolute last verified data frame configuration.
* Increments an error cycle counter; if failures exceed a strict limit threshold, the board executes a managed safe shutdown sequence.

---

## 8. Multi-Source Optimization Compatibility

Manifold edge firmware nodes **never interpret** complex parameters regarding *Extra Heating* (solar surplus) or biomass interactions. Peripherals focus exclusively on querying the unified single-row cache layout inside **`termostat_warming_request`**. All data integration and source blending math are isolated within the server layer.

---

## 9. Local Serial Telemetry Diagnostics
The edge firmware pushes structured diagnostic telemetry logs across the hardware UART serial interface, supporting custom verbose logging configurations:
* Exact raw SQL query transaction executions.
* Instantaneous local relay state matrix configurations.
* Direct MySQL connection error return codes.
* Measured mechanical stroke transition durations.
* Router Wi-Fi re-connection event records.
* Local internal system runtime clock tracking.
