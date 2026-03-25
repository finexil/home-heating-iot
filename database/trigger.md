# Trigger del Database – Home Heating IoT

I trigger del database MariaDB sono un elemento fondamentale dell’architettura del sistema.
Servono a mantenere sincronizzate le tabelle “now”, evitando che dispositivi a bassa potenza
(ESP8266/ESP32) debbano eseguire query pesanti o complesse.

Grazie ai trigger:

- ogni termostato invia solo i dati **minimali**, e il DB si occupa della logica;
- gli attuatori leggono dati già “pronti”, sempre aggiornati;
- la caldaia/termostufa non devono eseguire join o query multiple;
- tutto il sistema rimane coerente e reattivo.

---

# 1. Trigger su `Warming_state` → aggiorna `termostat_warming_request`

Ogni volta che un termostato invia un nuovo stato (ON/OFF) nella tabella `Warming_state`,
un trigger aggiorna la tabella `termostat_warming_request`, che contiene l’ultimo valore
per ciascuna zona.

Questa tabella viene letta dai controller delle elettrovalvole.

## ✅ Logica del trigger

- Identificare la `room` (es. “pt soggiorno”, “p1 camera”, ecc.)
- Aggiornare il relativo campo nella riga di `termostat_warming_request`
- Non inserire nuove righe, ma aggiornare sempre la stessa (location=“ceresole”)

## ✅ Definizione trigger Update_termostat_warming_request

```sql
CREATE TRIGGER Update_termostat_warming_request
AFTER INSERT ON Warming_state
FOR EACH ROW
BEGIN
    IF NEW.room = 'pt soggiorno' THEN
        UPDATE termostat_warming_request
        SET `pt_soggiorno` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'pt camera' THEN
        UPDATE termostat_warming_request
        SET `pt_camera` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'pt bagno' THEN
        UPDATE termostat_warming_request
        SET `pt_bagno` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 soggiorno' THEN
        UPDATE termostat_warming_request
        SET `p1_soggiorno` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 camera' THEN
        UPDATE termostat_warming_request
        SET `p1_camera` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 bagno' THEN
        UPDATE termostat_warming_request
        SET `p1_bagno` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'te terrazzo' THEN
        UPDATE termostat_warming_request
        SET `te_terrazzo` = NEW.FLG_ON
        WHERE location = 'ceresole';
    END IF;

END;

---

# 2. Trigger su `external_temp_hum` → `aggiorna external_temp_hum_now`

Il sensore esterno invia una riga ogni 10 minuti.
Il trigger aggiorna la tabella “now”, così i dispositivi che leggono il meteo esterno
eseguono una sola query.

Questa tabella viene letta da tutti i termostati.

## ✅ Logica del trigger

- Legge l'ultimo valore memorizzato nel db in termini di temperatura, umidità e pressione
lo memorizza nella tabella external_temp_hum_now

## ✅ Definizione trigger Update_external_temp_hum_now

```sql
CREATE TRIGGER Update_external_temp_hum_now
AFTER INSERT ON external_temp_hum
FOR EACH ROW
BEGIN
    UPDATE external_temp_hum_now
    SET
        `temperature` = NEW.temperature,
        `humidity`    = NEW.humidity,
        `pressure`    = 0
    WHERE location = 'ceresole';
END;

---

# 3. Trigger su `powerDetails` → `aggiorna termostat_temp_now`

La tabella powerDetails (nome storico) riceve i dati temperatura/umidità dei termostati.
Il trigger aggiorna immediatamente i valori nella tabella “now”, così l’interfaccia web del Raspberry
e gli algoritmi centrali leggono la temperatura “fresca”.

Questa tabella viene letta dai termostati con ESP32 e video grande per la pagina di dettaglio dello stato
di riscaldamento delle stanze con l'ultima temperatura rilevata, nonchè dalle logiche di analisi del dato
attive sul raspberry.

## ✅ Logica del trigger

- L'aggiornamento della tabella powerDetails aggiorna termostat_temp_now con il valore della stanza (room)
e temperatura con la verifica che la location sia quella corretta.

## ✅ Definizione trigger powerDetails_after_insert
```sql
CREATE TRIGGER powerDetails_after_insert
AFTER INSERT ON powerDetails
FOR EACH ROW
BEGIN

    IF NEW.room = 'pt soggiorno' THEN
        UPDATE termostat_temp_now
        SET `pt soggiorno` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'pt camera' THEN
        UPDATE termostat_temp_now
        SET `pt camera` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'pt bagno' THEN
        UPDATE termostat_temp_now
        SET `pt bagno` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 soggiorno' THEN
        UPDATE termostat_temp_now
        SET `p1 soggiorno` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 camera' THEN
        UPDATE termostat_temp_now
        SET `p1 camera` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'p1 bagno' THEN
        UPDATE termostat_temp_now
        SET `p1 bagno` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

    IF NEW.room = 'te terrazzo' THEN
        UPDATE termostat_temp_now
        SET `te terrazzo` = NEW.temperature
        WHERE location = 'ceresole';
    END IF;

END;

---

# 4. Perché i trigger sono fondamentali?
✅ Riduzione del traffico di rete
Gli ESP8266/ESP32 leggono una sola riga invece di centinaia.
✅ Database sempre pulito e coerente
Una sola tabella contiene lo stato aggiornato.
✅ Logica centralizzata
Il logic “who controls what” è nel Raspberry, non nei dispositivi.
✅ Aggiornamenti immediati
Ogni variazione di un termostato si propaga istantaneamente a tutta l’architettura.

---

# 5. Diagramma del ciclo dei trigger
[Termostato] 
   → INSERT in Warming_state
      → Trigger → aggiorna termostat_warming_request

[Termostato]
   → INSERT in powerDetails
      → Trigger → aggiorna termostat_temp_now

[Sensore esterno]
   → INSERT in external_temp_hum
      → Trigger → aggiorna external_temp_hum_now

---

# 6. File correlati

schema.md → descrizione delle tabelle
overview.md → visione d’insieme del DB
devices.md → come i dispositivi usano i trigger
algorithms/extra_heating.md → come gli algoritmi leggono le tabelle “now”

