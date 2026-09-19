markdown
# Database Triggers (SQL DDL)
🇮🇹 *Per la versione in italiano, clicca [qui](triggers.it.md).*

To minimize network traffic and remove heavy computational strain from low-power ESP8266 and ESP32 edge nodes, this architecture leverages native **MariaDB Server-Side Triggers**. 

Upon any time-series data stream insertion, these triggers instantly execute to keep single-row cache lookup tables up to date. This allows edge nodes to fetch system-wide states instantly using lightweight, flat row queries.

---

## 🛠️ Production Trigger Definitions

### 1. Update Thermostat Warming Request
* **Source Table:** `Warming_state`
* **Target Cache Table:** `termostat_warming_request`
* **Execution Event:** `AFTER INSERT`
* **Purpose:** Maps relational event-driven zone requirements directly into a flat column layout for immediate polling by manifold controllers.

```sql
DELIMITER \$\$

CREATE TRIGGER `Update_termostat_warming_request` 
AFTER INSERT ON `Warming_state`
FOR EACH ROW
BEGIN
    IF NEW.room = 'pt soggiorno' THEN
        UPDATE termostat_warming_request SET `pt_soggiorno` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'pt camera' THEN
        UPDATE termostat_warming_request SET `pt_camera` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'pt bagno' THEN
        UPDATE termostat_warming_request SET `pt_bagno` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 soggiorno' THEN
        UPDATE termostat_warming_request SET `p1_soggiorno` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 camera' THEN
        UPDATE termostat_warming_request SET `p1_camera` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 bagno' THEN
        UPDATE termostat_warming_request SET `p1_bagno` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'te terrazzo' THEN
        UPDATE termostat_warming_request SET `te_terrazzo` = NEW.FLG_ON WHERE location = 'ceresole';
    END IF;
END\$\$

DELIMITER ;
```

### 2. Update Live Outdoor Conditions
* **Source Table:** `External_temp_hum`
* **Target Cache Table:** `external_temp_hum_now`
* **Execution Event:** `AFTER INSERT`
* **Purpose:** Pushes the latest outdoor environmental metrics into a high-speed lookup cache for local thermostat display panels.

```sql
DELIMITER \$\$

CREATE TRIGGER `Update_external_temp_hum_now` 
AFTER INSERT ON `External_temp_hum`
FOR EACH ROW
BEGIN
    UPDATE external_temp_hum_now 
    SET `temperature` = NEW.temperature, 
        `humidity` = NEW.humidity, 
        `pressure` = 0 
    WHERE location = 'ceresole';
END\$\$

DELIMITER ;
```

### 3. Update Real-Time Room Telemetry
* **Source Table:** `powerDetails`
* **Target Cache Table:** `termostat_temp_now`
* **Execution Event:** `AFTER INSERT`
* **Purpose:** Automatically caches the most recent indoor room temperatures so secondary display nodes can monitor other areas of the house instantly.

```sql
DELIMITER \$\$

CREATE TRIGGER `powerDetails_after_insert` 
AFTER INSERT ON `powerDetails`
FOR EACH ROW
BEGIN
    IF NEW.room = 'pt soggiorno' THEN
        UPDATE termostat_temp_now SET `pt soggiorno` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'pt camera' THEN
        UPDATE termostat_temp_now SET `pt camera` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'pt bagno' THEN
        UPDATE termostat_temp_now SET `pt bagno` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 soggiorno' THEN
        UPDATE termostat_temp_now SET `p1 soggiorno` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 camera' THEN
        UPDATE termostat_temp_now SET `p1 camera` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'p1 bagno' THEN
        UPDATE termostat_temp_now SET `p1 bagno` = NEW.temperature WHERE location = 'ceresole';
    END IF;
    IF NEW.room = 'te terrazzo' THEN
        UPDATE termostat_temp_now SET `te terrazzo` = NEW.temperature WHERE location = 'ceresole';
    END IF;
END\$\$

DELIMITER ;
```
