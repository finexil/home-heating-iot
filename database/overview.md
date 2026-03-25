# Overview del Database – Home Heating IoT

Questo documento fornisce una panoramica architetturale del database MariaDB utilizzato
dal sistema Home Heating IoT.  
È una visione d’insieme che completa i file:

- `schema.md` → dettagli delle tabelle  
- `triggers.md` → funzionamento dei trigger  
- `devices.md` → come i dispositivi usano il DB

L'obiettivo è capire **come circolano i dati** all’interno del sistema e **perché il DB è il cuore dell’intera automazione**.

---

# 1. Architettura generale del database

Il sistema utilizza due database distinti:

- **`temperature`** → logica del riscaldamento e stato delle zone  
- **`weather`** → previsioni meteo e dati esterni

I termostati, gli attuatori e i controller comunicano solo con `temperature`.
Il server Python del Raspberry Pi utilizza entrambi.

L’architettura è stata progettata per:

✅ minimizzare il carico sugli ESP8266/ESP32  
✅ garantire aggiornamenti immediati tramite trigger  
✅ fornire dati “puliti” e subito disponibili  
✅ registrare uno storico utile per analisi e algoritmi adattativi

---

# 2. Ruolo delle tabelle principali

### **2.1 Tabelle “storiche”**
Sono tabelle che crescono nel tempo:

- `Warming_state` → eventi ON/OFF dei termostati  
- `termostat_temp_hum` → temperature/umidità (ogni 10 min)  
- `external_temp_hum` → meteo esterno (ogni 10 min)  
- `zone_status` → stato valvole (solo se cambia)  
- `heating_state` → stato caldaia/pompa

Queste tabelle permettono:

✅ analisi a lungo termine  
✅ grafici  
✅ algoritmi adattativi basati sul comportamento reale dell’impianto  

---

### **2.2 Tabelle “NOW”**
Sono tabelle a una sola riga, usate per consultazione ISTANTANEA:

- `termostat_warming_request`  
- `termostat_temp_now`  
- `external_temp_hum_now`

Queste tabelle esistono per un motivo fondamentale:

✅ gli ESP8266/ESP32 leggono una sola riga  
✅ niente join, niente query pesanti  
✅ massima stabilità con WiFi intermittente  

Le tabelle “NOW” vengono aggiornate automaticamente dai **trigger**.

---

# 3. Flusso dei dati all’interno del sistema

Il flusso dati si può riassumere in 4 cicli principali.

---

## **3.1 Ciclo Termostati → DB**
Ogni termostato:

1. ogni 5 secondi misura temperatura/umidità  
2. ogni 10 minuti inserisce nella tabella `termostat_temp_hum`  
3. quando cambia ON/OFF inserisce in `Warming_state`  
4. aggiorna periodicamente la configurazione letta dal DB  

Grazie ai trigger:

- `Warming_state` → `termostat_warming_request`  
- `termostat_temp_hum` → `termostat_temp_now`

I termostati **NON** dialogano direttamente con gli attuatori, né con la caldaia.

---

## **3.2 Ciclo Attuatori → DB**
Ogni attuatore:

1. legge **solo** la tabella `termostat_warming_request`  
2. attiva/disattiva elettrovalvole  
3. aggiorna lo stato attuale in `zone_status`

Gli attuatori non fanno calcoli e non hanno logiche complesse.

---

## **3.3 Ciclo Caldaia / Termostufa → DB**
Il controller della caldaia:

1. legge `zone_status`  
2. consulta `heating_state`  
3. decide accensione/spegnimento  
4. registra ciò che fa in `heating_state`

Il controller della termostufa:

1. rileva il contatto di acqua calda  
2. aggiorna `heating_state`  
3. forza tutte le zone in riscaldamento (eccetto >22°C)

---

## **3.4 Ciclo Algoritmi → DB**
Gli algoritmi Python sul Raspberry Pi:

- leggono continuamente `termostat_temp_now`  
- leggono `external_temp_hum_now`  
- analizzano i consumi e la produzione fotovoltaica via API  
- aggiornano `heating_state` per Extra Heating  
- aggiornano configurazioni orarie adattive

Sono gli unici a usare tutto il database in profondità.

---

# 4. Diagramma completo del flusso dati
            ┌────────────────────┐
            │     Termostati     │
            │ ESP8266 / ESP32    │
            └───────┬────────────┘
                    │ INSERT
                    ▼
            ┌───────────────────┐
            │   Warming_state   │──────┐
            └───────────────────┘      │ trigger
                                       ▼
                            ┌────────────────────────────┐
                            │ termostat_warming_request  │
                            └────────────────────────────┘
                                       │
                                       ▼
                            ┌────────────────────────────┐
                            │   Attuatori elettrovalvole │
                            └──────────┬─────────────────┘
                                       │ UPDATE
                                       ▼
                          ┌──────────────────────┐
                          │     zone_status      │
                          └──────────┬───────────┘
                                     │
                                     ▼
                   ┌─────────────────────────┐
                   │ Caldaia / Termostufa    │
                   └──────────┬──────────────┘
                              │ WRITE
                              ▼
                   ┌─────────────────────────┐
                   │     heating_state       │
                   └──────────┬──────────────┘
                              │
                              ▼
                 ┌───────────────────────────────┐
                 │   Algoritmi Raspberry Pi      │
                 └───────────────────────────────┘


---

# 5. Motivazioni progettuali

### ✅ **Aver scelto un DB relazionale centralizzato permette:**
- sincronizzazione automatica tra tutti i componenti  
- logiche complesse senza caricare gli ESP  
- possibilità di analisi avanzate nel tempo  
- controllo totale del sistema senza servizi cloud  

### ✅ **Alternative (MQTT, Home Assistant, Node-RED) erano possibili ma…**
Il tuo sistema richiede:

- sincronizzazione millimetrica tra zone  
- logica deterministica  
- storico centralizzato  
- algoritmi personalizzati

La scelta di MariaDB risponde a tutti questi requisiti.

---

# 6. Conclusioni

Questa overview sintetizza:

- il ruolo del DB nel sistema  
- il ciclo completo dei dati  
- l’interazione tra dispositivi, trigger e algoritmi  
- le motivazioni tecniche che giustificano l’architettura

Per approfondire:

- vedi `schema.md` → dettaglio tabelle  
- vedi `triggers.md` → funzionamento trigger  
- vedi `devices.md` → uso del DB da parte degli apparati


