# Controller Caldaia + Pompa di Ricircolo – Firmware ESP8266

Il controller caldaia gestisce l’accensione della pompa di ricircolo e della caldaia elettrica
in base allo stato delle zone e alle condizioni dell’impianto.

Questa MCU è una delle più critiche dell’intero sistema, poiché garantisce:

✅ continuità del riscaldamento  
✅ protezione della caldaia  
✅ coordinamento con termostufa e algoritmi centrali  
✅ gestione cicli ON/OFF controllati  
✅ funzionamento sicuro anche con WiFi intermittente  

---

# 1. Architettura generale

Il controller:

- si connette al WiFi locale  
- apre una connessione diretta a MariaDB (libreria MySQL_MariaDB_Generic)  
- legge lo stato aggregato delle zone (`zone_status`)  
- legge lo stato della termostufa e dell’extra heating (`heating_state`)  
- decide se attivare:
  - 🔹 la **pompa di ricircolo**  
  - 🔹 la **caldaia elettrica**  

- applica ritardi per protezione meccanica  
- registra log nella tabella `heating_state`

L’intera logica è centralizzata nel DB → il controller è **stateless**.

---

# 2. Hardware

- MCU: **ESP8266 NodeMCU 12-E 1.0 Module**
- Relè (o SSR) per:
  - 🔹 caldaia  
  - 🔹 pompa di ricircolo  
- Alimentazione filtrata con step-down e stabilizzatore
- Possibile sensore temperatura acqua (opzionale, usato dalla termostufa)

---

# 3. Connessione MySQL – Architettura

er tutte le MCU basate su ESP8266, la connessione al DB avviene tramite la libreria ![MySQL_MariaDB_Generic](https://github.com/khoih-prog/MySQL_MariaDB_Generic).

Credenziali dedicate con permessi:

-SELECT
-INSERT → per log funzionamento

Non sono concessi privilegi di creazione/modifica tabelle.

# 4. Logica del ciclo operativo
Il controller esegue un ciclo continuo ogni 90 secondi:
LOOP PRINCIPALE
    → SELECT su zone_status
    → Verifica se almeno una zona è attiva
    → INSERT su heating_state per tutte le azioni intraprese
    → Determina azione:

         - solo pompa
         - pompa + caldaia
         - tutto spento

    → Rispetta ritardi meccanici (ritardo caldaia)
    → Aggiorna log in heating_state


# 5. Query SQL utilizzate

## 🔍 5.1 Lettura delle zone attive

```sql
SELECT  pt_soggiorno, pt_camera, pt_bagno,  p1_soggiorno, p1_camera, p1_bagno,  te_terrazzo
FROM zone_status
WHERE location='ceresole';
```

## 🔍 5.2 Registrazione log funzionamento caldaia/pompa

```sql
INSERT INTO temperature.heating_state
(distributor_zone, heating_status, reason, pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo, termostufa)
VALUES ('%s', %d, 'HEATING OFF DUE TO TERMOSTATS OR STOVE REQUEST', %d, %d, %d, %d, %d, %d, %d, %d)";
,room, heating_status, zone_status[1], zone_status[2], zone_status[3], zone_status[4], zone_status[5], zone_status[6], zone_status[7], zone_status[8]
```

# 6. Logica di accensione caldaia e pompa
## 6.1 ✅ Caso 1 — Almeno una zona attiva
→ Accende la pompa
→ Attende ~30 secondi
→ Accende la caldaia

## 6.2 ✅ Caso 2 — Termostufa attiva
→ Accende la pompa
→ Attende ~30 secondi
→ Accende la caldaia

Se la temperatura dell’acqua del serbatoio della caldaia supera la soglia impostata nella sua configurazione (nel mio caso 49°C), in automatico la pompa di calore viene fermata.

## 6.3 ✅ Caso 3 — Extra Heating attivo
Il riscaldamento è richiesto da tutte le zone (da tutti i termostati):
→ Accende la pompa
→ Attende ~30 secondi
→ Accende la caldaia

I termostatati attivano la zona solo se la temperatura del locale è inferiore a 22°C, in pratica se l'Extra Heating si attivasse in una giornata calda con stanze riscaldate dal sole che raggiungono, seppure in periodo freddo, la temperatura di 22°C, i termostati non attivano la caldaia.

## 6.4 ✅ Caso 4 — Nessuna zona attiva
→ Spegne la caldaia
→ Attende 30 secondi
→ Spegne la pompa


# 7. Sicurezze e timeout
✅ Ritardo ON caldaia
Evita accensioni quando tutte le valvole sono ancora chiuse.
✅ Ritardo OFF pompa
Evita shock termici.
✅ Timeout totale accensione
Se la caldaia rimane ON troppo a lungo senza zone davvero aperte:
→ spegnimento completo
→ log evento

## 7.1 ✅ Recupero dopo perdita WiFi

mantiene lo stato corrente
tenta riconnessione
dopo 30s senza connessione:
- reboot dell'MCU e reset Relè
- ripristino condizione con lettura DB e ciclo LOOP


## 7.2 ✅ Protezione mancanza stato coerente
Se zone_status non restituisce valori coerenti:
→ spegnimento totale
→ log in heating_state


# 8. Routing logico della termostufa
La termostufa ordina lo stato ai termostati che portano all'accensione della caldaia:

se la stufa fornisce acqua >60°C (temperatura serbatoio caldaia > 49°C), la pompa di calore si ferma
se la stufa è spenta → normale logica delle zone e la pompa di calore è in funzione

La termostufa “vince” sempre sulla pompa di calore per priorità termica.

# 9. Logging e diagnostica
Il firmware registra via SQL e seriale:

stato caldaia
stato pompa
stato termostufa
errori MySQL
riconnessioni WiFi
tempi di attivazione/disattivazione
modalità trigger di Extra Heating

La tabella heating_state consente uno storico completo, usato anche dagli algoritmi, la verbosità del log da seriale è configurabile.

# 10. Estensioni future possibili

sensore portata acqua
rilevazione anomalie (temperatura troppo alta)
integrazione con algoritmo anti‑legionella
ottimizzazione ritardi con auto‑apprendimento
gestione manuale tramite interfaccia web locale
