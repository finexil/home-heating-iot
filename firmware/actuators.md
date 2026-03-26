# Controller Attuatori Elettrovalvole – Firmware ESP8266

Questo documento descrive il firmware dei controller basati su **ESP8266 Lolin D1 Mini Lite**
che pilotano le elettrovalvole del riscaldamento a pavimento.  
Ogni dispositivo gestisce da 2 a 4 zone a seconda della centralina.

Gli attuatori sono responsabili della **parte fisicamente attiva dell’impianto**:
aprono e chiudono le elettrovalvole in base alle richieste provenienti dai termostati.

---

# 1. Architettura generale

Ogni controller:

✅ si connette al WiFi locale  
✅ apre una connessione diretta **MySQL/MariaDB** al Raspberry Pi  
✅ esegue query SQL per leggere le richieste delle zone  
✅ pilota i relè delle elettrovalvole  
✅ aggiorna lo stato effettivo (`zone_status`)  
✅ applica timeout e protezioni meccaniche  
✅ continua a funzionare anche in caso di brevi dropout WiFi  

Gli attuatori sono **stateless** lato firmware: lo stato finale dipende sempre dal DB.

---

# 2. Hardware

- MCU: **ESP8266 D1 Mini Lite**
- Relè 230V o 12V collegati tramite transistor e diodi di protezione
- Alimentazione: 12V + step-down +5V → 3.3V per ESP
- LED diagnostici (opzionali)
- Circuiti identici nelle centraline Piano Terra e Primo Piano

---

# 3. Connessione Diretta MySQL (importante)

Gli attuatori **NON usano HTTP/PHP**, ma si connettono **direttamente** al server MariaDB.

Libreria tipica:
MySQL_Connection conn((Client*)&wifi);
MySQL_Cursor cursor(&conn);

All’avvio:

1. connessione al WiFi  
2. connessione TCP al server MariaDB  
3. autenticazione con **utente dedicato** che ha solo privilegi:
   - `SELECT`  
   - `UPDATE`  
   - (eventualmente `INSERT` per log)  

L’utente NON ha alcun privilegio di alterazione o lettura di tabelle sensibili.

---

# 4. Flusso logico del firmware

Ogni attuatore esegue un ciclo:


LOOP PRINCIPALE (ogni 5–10 secondi)
→ SELECT su termostat_warming_request
→ Identifica le zone gestite da questo controller
→ Determina ON/OFF richiesto
→ Applica timeout meccanico
→ Attiva/Disattiva relè
→ UPDATE di zone_status
→ Aggiorna timestamp locale

---

# 5. Query SQL utilizzate

## 🔍 5.1 Lettura stato richiesto dai termostati (tabella termostat_warming_request)

```sql
SELECT pt_soggiorno, pt_camera, pt_bagno
FROM termostat_warming_request
WHERE location='ceresole';
```

(I campi delle stanze cambiano per il controller del primo piano)

## 🔍 5.2 Lettura stato delle zone impostate nel DB (tabella zone_status)

```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo
FROM temperature.zone_status WHERE location='ceresole';
```

## 🔍 5.3 Aggiornamento dello stato effettivo

Update per lo stato OFF:

```sql
UPDATE temperature.zone_status
SET pt_soggiorno=0, pt_camera=0, pt_bagno=0 WHERE location='ceresole';
```

Update per lo stato ON:

```sql
UPDATE temperature.zone_status
SET pt_soggiorno=1, pt_camera=1, pt_bagno=1 WHERE location='ceresole';
```

(I campi delle stanze cambiano per il controller del primo piano)

## 🔍 5.4 Aggiornamento tabella stati (heating_status)

Questa tabella contiene il log delle attivazioni delle zone da parte dei controller

```sql
INSERT INTO temperature.heating_state
(distributor_zone, heating_status, reason, pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo)
 VALUES ('%s', %d, '##stato', %d, %d, %d, %d, %d, %d, %d);",
zona, heating_status, zone_status[1], zone_status[2], zone_status[3], zone_status[4], zone_status[5], zone_status[6], zone_status[7]
```

# 6. Gestione dei timeout
Ogni elettrovalvola necessita di:

tempo minimo di apertura (~90s)
tempo minimo di chiusura (~90s)
protezioni contro cicli rapidi ON/OFF

Ogni elettrovalvola impiega fino a 3' (180s) per aprire/chiudere completamente il rubinetto, 
la zona viene pertanto considerata attiva o disattiva quando l'elettrovalvola è almeno a metà
del ciclo, così da non dare il via alla pompa di ricircolo della caldaia con impianti ancora chiusi.

Il firmware prevede:
✅ timer software indipendenti per ogni zona
✅ ritardi minimi configurabili
✅ blocco temporaneo in caso di oscillazioni rapide
✅ fallback se lo stato cambia troppo frequentemente

# 7. Fallback in caso di problemi
L’attuatore ha meccanismi di sicurezza se:
✅ il WiFi cade

riprova connessione ogni 5 secondi
reboot dell'MCU dopo timeout di 30s, ripristino dello stato ad OFF degli attuatori sia tramite i relè sia aggiornando il DB
ripristinato il ciclo di loop, con la lettura del DB viene ripristinato lo stato richiesto
evita chiusure/aperture incontrollate

✅ MySQL non risponde

log interno via seriale
retry connessione
nessuna variazione dello stato finché non è certa la richiesta

✅ lettura SQL fallisce

fallback su ultimo stato valido
timeout dopo N fallimenti → spegnimento controllato


# 8. Compatibilità con Extra Heating e Termostufa
Gli attuatori NON interpretano extra heating o termostufa.
Leggono semplicemente lo stato impostato dai termostati hella tabella:

termostat_warming_request


# 9. Logging (via seriale)
Il firmware registra:

query SQL eseguite
stato zone
errori MySQL
tempi di apertura/chiusura
riconnessioni WiFi
timestamp locale

Questi log sono utili per debug e diagnostica e configurabili nella verbosità.

10. Estensioni future possibili

watchdog hardware
sincronizzazione con timestamp del server
controllo alimentazione relè via triac o SSR
sistema di auto-diagnostica per valvole bloccate
invio log al server via tabelle dedicate (facoltativo)
