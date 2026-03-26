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

L’ESP8266 usa la libreria MySQL_Generic.h di MySQL_MariaDB_Generic

Credenziali dedicate con permessi:

-SELECT
-INSERT → per log funzionamento
-UPDATE → eventuali note operative

Non sono concessi privilegi di creazione/modifica tabelle.

4. Logica del ciclo operativo
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


5. Query SQL utilizzate
🔍 5.1 Lettura delle zone attive


SQLSELECT  pt_soggiorno, pt_camera, pt_bagno,  p1_soggiorno, p1_camera, p1_bagno,  te_terrazzoFROM zone_statusWHERE location='ceresole';Mostra più linee
🔍 5.2 Lettura stato termostufa / extra heating
SQLSELECT stove_on, extra_heating_onFROM heating_stateORDER BY timestamp DESCLIMIT 1;Mostra più linee
🔍 5.3 Registrazione log funzionamento caldaia/pompa
SQLINSERT INTO heating_state(pump_on, boiler_on, stove_on, timestamp)VALUES (%d, %d, %d, NOW());Mostra più linee

7. Logica di accensione caldaia e pompa
✅ Caso 1 — Almeno una zona attiva
→ Accende la pompa
→ Attende ~30 secondi
→ Accende la caldaia

✅ Caso 2 — Termostufa attiva
Se la temperatura dell’acqua è sopra soglia:
→ Attiva solo la pompa
→ NON accende la caldaia

✅ Caso 3 — Extra Heating attivo
Il riscaldamento è richiesto da tutte le zone MA:

si attivano gli attuatori prima
la caldaia parte solo se almeno una zona rimane sotto soglia → logica conservativa

✅ Caso 4 — Nessuna zona attiva
→ Spegne la caldaia
→ Attende 20–30 secondi
→ Spegne la pompa


7. Sicurezze e timeout
✅ Ritardo ON caldaia
Evita accensioni quando tutte le valvole sono ancora chiuse.
✅ Ritardo OFF pompa
Evita shock termici.
✅ Timeout totale accensione
Se la caldaia rimane ON troppo a lungo senza zone davvero aperte:
→ spegnimento completo
→ log evento

✅ Recupero dopo perdita WiFi

mantiene lo stato corrente
tenta riconnessione
dopo 30s senza connessione:

spegne caldaia
spegne pompa
esegue reboot



✅ Protezione mancanza stato coerente
Se zone_status non restituisce valori coerenti:
→ spegnimento totale
→ log in heating_state


8. Routing logico della termostufa
Il controller caldaia si coordina con la termostufa:


se la stufa fornisce acqua >60°C:

stove_on = 1
caldaia OFF
pompa ON

se la stufa è spenta → normale logica delle zone


La termostufa “vince” sempre sulla caldaia per priorità termica.

9. Logging e diagnostica
Il firmware registra via SQL e seriale:

stato caldaia
stato pompa
stato termostufa
errori MySQL
riconnessioni WiFi
tempi di attivazione/disattivazione
modalità trigger di Extra Heating

La tabella heating_state consente uno storico completo, usato anche dagli algoritmi.

10. Estensioni future possibili

sensore portata acqua
rilevazione anomalie (temperatura troppo alta)
integrazione con algoritmo anti‑legionella
ottimizzazione ritardi con auto‑apprendimento
gestione manuale tramite interfaccia web locale
