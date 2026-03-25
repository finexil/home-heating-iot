# Overview del Firmware – Home Heating IoT

Questa sezione descrive l’architettura e la logica generale del firmware utilizzato
nei dispositivi ESP8266 ed ESP32 del sistema Home Heating IoT.

Il codice è organizzato in base ai ruoli:

- **Termostati (ESP8266 / ESP32)**: misurano temperatura, visualizzano interfaccia,
  inviano richieste di riscaldamento e aggiornano il DB.
- **Attuatori (ESP8266)**: leggono lo stato dal DB e pilotano elettrovalvole.
- **Controller caldaia/pompa**: attivano la generazione del calore in base alle zone attive.
- **Controller termostufa**: rilevano l’acqua calda e aggiornano lo stato del sistema.

---

# 1. Architettura del firmware

Tutti i firmware condividono alcune caratteristiche:

✅ WiFi always-on con riconnessione automatica  
✅ Comunicazione HTTP verso il Raspberry Pi  
✅ Query semplici e ottimizzate  
✅ Logging essenziale per debug  
✅ Watchdog attivo per evitare blocchi  
✅ Boot rapido con autocontrollo dei sensori  

Negli ESP32 è presente anche:

✅ FreeRTOS con task multipli  
✅ Display touch con aggiornamento parallelo  
✅ Gestione sensori multipli (CO₂, TVOC, temperatura, umidità)  

---

# 2. Comunicazione con il server

La comunicazione tra ESP e Raspberry Pi avviene tramite:

- richieste HTTP GET/POST
- endpoint PHP che eseguono query SQL
- parametri semplici (es: `room=pt_soggiorno&temp=21.5`)
- risposta JSON o testo semplice

Esempi:
/update_temp.php?room=pt_soggiorno&temp=21.3&hum=45

/get_warming_state.php?room=pt_soggiorno

---

# 3. Formato dati scambiati

I termostati inviano:

- temperatura media degli ultimi 10 minuti  
- stato richiesto (ON/OFF)  
- forzatura (NORMAL/FORCE ON/FORCE OFF)  
- modalità attiva (configurazione / extra heating / stufa)  
- data e ora locale  

Gli attuatori leggono:

- `termostat_warming_request` (una sola riga)  
- eventuali extra flag da `heating_state`

---

# 4. Logica di funzionamento dei dispositivi

## 4.1 Termostati ESP8266
- Lettura sensore temperatura/umidità  
- Visualizzazione display 1.8”  
- Invio temperatura ogni 10 minuti  
- Invio stato riscaldamento quando cambia  
- Calcolo media sulle ultime letture  
- Gestione pulsanti (pagina e forzatura)  

---

## 4.2 Termostati ESP32 con FreeRTOS
Il firmware è diviso in più task:

- **task_sensori** (alta priorità)  
- **task_display** (media priorità)  
- **task_net** (bassa priorità)  
- **task_logiche** (media priorità)

Questa divisione garantisce:

✅ UI fluida  
✅ letture stabili  
✅ nessun blocco WiFi  
✅ comportamento deterministico  

---

## 4.3 Attuatori delle elettrovalvole (ESP8266)
- Richiedono `termostat_warming_request`  
- Attivano/disattivano relè  
- Segnalano lo stato effettivo  
- Applicano timeout per proteggere elettrovalvole e caldaia  

---

## 4.4 Controller Caldaia
- Accende la pompa al bisogno  
- Accende la caldaia dopo ritardo  
- Legge `heating_state` e `zone_status`  
- Implementa logica di sicurezza per cicli ON/OFF  

---

## 4.5 Controller Termostufa
- Legge sensore temperatura acqua  
- Aggiorna `heating_state`  
- Forza tutte le zone  
- Previene sovratemperature nelle stanze (>22°C)

---

# 5. File specifici del firmware

I file di dettaglio sono:

- `thermostat_esp8266.md`  
- `thermostat_esp32.md`  
- `actuators.md`  
- `boiler_controller.md`  
- `stove_controller.md`

Questi file descrivono:
- pinout  
- logica completa  
- pseudocodice  
- diagrammi  
- routine principali  

---

# 6. Obiettivo della documentazione

Questa documentazione serve per:

✅ comprendere l’architettura generale del firmware  
✅ rendere più semplice la manutenzione  
✅ facilitare chi vuole prendere spunto dal progetto  
✅ separare la logica dalla parte implementativa (codice reale)
