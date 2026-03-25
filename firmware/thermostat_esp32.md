# Termostato ESP32 con display 3.5" Touch – Dettagli del Firmware

Questo documento descrive in dettaglio il firmware del termostato basato su **ESP32**
con display TFT **3.5" touch**.  
È la versione più avanzata del sistema, utilizzata nelle stanze principali (soggiorni, terrazzo).

Il firmware sfrutta **FreeRTOS**, più sensori, grafica avanzata, pagine multiple
e interazioni complesse con il server.

---

# 1. Architettura generale del firmware

Obiettivi principali:

✅ UI fluida con touchscreen  
✅ lettura sensori in tempo reale  
✅ comunicazione affidabile con il server  
✅ grafici e dati aggiornati  
✅ gestione logiche locali + icone di stato  
✅ task separati per garantire stabilità  

L’ESP32 permette di isolare ogni componente in un **task dedicato**, evitando blocchi del display o ritardi di rete.

---

# 2. Componenti Hardware

- **ESP32-WROOM / ESP32-DevKit**
- Display **TFT 3.5" SPI** con touch resistivo (driver XPT2046)
- Sensori:
  - **BME280**: temperatura / umidità / pressione
  - **CCS811**: qualità aria (TVOC/CO₂ stimato)
  - regolatore LED display per spegnimento automatico dopo 1' e 30"
- Alimentazione: 5V → regolazione a 3.3V tramite l'MCU, uscita 3,3V
- Clock: 240 MHz

---

# 3. Pinout consigliato

### Display (ST7796 / ILI9488)

| Funzione | ESP32 Pin |
|----------|-----------|
| CS       | 5         |
| RESET    | 4         |
| DC       | 2         |
| MOSI     | 23        |
| MISO     | N.C.      |
| SCK      | 18        |
| LED      | 16*       |

* Il pin 16 pilota un transistor e non direttamente il LED
  
### Touch (XPT2046)

| Funzione | ESP32 Pin |
|----------|-----------|
| T_CS     | 15        |
| T_CLK    | 18        |
| T_DIN    | 23        |
| T_D0     | 19        |
| T_IRQ    | N.C.      |

### Sensori

| Sensore | Pin | Address |
|---------|-----|---------|
| BME280 (I2C) | SDA=21, SCL=22 | 0x77, 0x76 | 
| CCS811 (I2C) | SDA=21, SCL=22, WAC=GND| 0x5A |

---

# 4. FreeRTOS – struttura dei task

Il firmware è strutturato su più task paralleli:

## ✅ 4.1 task_sensori (alta priorità)
- legge temperatura, umidità, pressione  
- legge qualità aria  
- applica filtri e stabilizzazione valori  
- aggiorna variabili condivise thread-safe  

Periodo: **100 ms**

---

## ✅ 4.2 task_DB (priorità media)
Gestisce tutte le chiamate al DB:

- INSERT 
- UPDATE  
- SELECT  
Le stringhe SQL sono memorizzate in una struttura condivisa tra i task

Periodo: **100**

---

## ✅ 4.3 task_Time (priorità media)
Gestisce date e ora nonchè il cambio ora legale/solare:

- Timer di innesco attività (1', 10', 24 ore)  
- Timer per lettura sensori ambiente  
- Timer per spegnimento automatico del LED Display dopo 1'30" di inattività  
- Timer aggiornamento dello stato e temperatura termostati dell'impianto   
- Timer aggiornamento potenza segnale Wifi
- Timer verifica connessione Wifi
- Timer aggiornamento previsioni
- Timer lettura Temperature orarie impostate nel DB per il termostato

---

## ✅ 4.4 task_Loop (default)
Gestisce la normale funzionalità del termostato, del display e del touch:

- Gestione stati e forzature 
- Gestione grafica del display  
- Aggiornamento previsioni Meteo 
- Aggiornamento Grafici  

Periodo: **100 ms**

---

# 5. Interfaccia Grafica (UI)

### ✅ Schermata principale
- Temperatura locale  
- Umidità  
- Qualità aria (barra CO₂/TVOC)  
- Icona riscaldamento (rossa/gialla/stufa)  
- Orario locale  
- Stato WiFi
- Temperatura esterna
- Temperatura oraria impostata
- Previsioni Meteo (delle prossime 6 ore)
- Qualità dell'aria

### ✅ Schermata grafico 24h
- grafico storico della temperatura e umidità locali rilevate e di quella esterna (ogni 10')  
- minima/massima  
- Grafico configurazione oraria del termostato con linea andamento temperatura ambiente

### ✅ Schermata stato zone
Vista con icone dello stato e della temperatura dei termostati dell'impianto

- icona termostato con colore rosso per riscaldamento attivo e verde per spento 
- temperatura della singola zona 

---

# 6. Logiche di riscaldamento

Il termostato ESP32 mostra la fonte del calore in modo dinamico:

| Icona | Significato |
|-------|-------------|
| <img width="40" height="39" alt="image" src="https://github.com/user-attachments/assets/d97f6b0c-e857-4e03-8112-6adf18e54451" /> | richiesta locale (configurazione oraria) |
| <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/6389740f-86ea-4227-b1d1-031ebf1e72d3" /> | extra heating da fotovoltaico |
| <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/53d6b121-8ec9-4bc0-9891-884fe79045de" /> | riscaldamento da termostufa |

Le logiche:

- **Extra Heating** e **Termostufa** forzano tutte le zone in riscaldamento  
- limite automatico a **22°C** per evitare surriscaldamento  
- il termostato mostra la modalità senza confusione

---

# 7. Comunicazione col server (DB)

Utilizzata per le board ESP32 la libreria ESP32_MySQL.h https://github.com/Syafiqlim/ESP32_MySQL
## ✅ Invio temperatura, umidità e pressione

sprintf(db_command[pt_stack_db_write].STRING_SQL, "INSERT INTO temperature.powerDetails (room, Temperature, Humidity, Pressure) VALUES ('%s', %.3f, %.3f, %.3f);", room[my_room_number], tf, hf, pf);

## ✅ Invio stato riscaldamento

sprintf(db_command[pt_stack_db_write].STRING_SQL, "INSERT INTO temperature.Warming_state (room, FLG_ON, FLG_FORCE) VALUES ('%s', %d, %d);", room[my_room_number], heating_status, heating_mode);

## ✅ Ricezione configurazione oraria

sprintf(db_command[pt_stack_db_write].STRING_SQL, "SELECT * FROM temperature.TermostatSetup where room='%s';", room[my_room_number]);

## ✅ Ricezione Stato e Temperatura Termostati

sprintf(db_command[pt_stack_db_write].STRING_SQL, "SELECT `pt soggiorno`, `pt camera`, `pt bagno`, `p1 soggiorno`, `p1 camera`, `p1 bagno`, `te terrazzo` FROM temperature.termostat_temp_now WHERE location='%s';", location);
sprintf(db_command[pt_stack_db_write].STRING_SQL, "SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo FROM temperature.zone_status WHERE location='%s';", location);

## ✅ Ricezione previsioni meteo

sprintf(db_command[pt_stack_db_write].STRING_SQL, "SELECT temperature, wind_speed, icon, description FROM weather.forecasts WHERE timestamp = (SELECT MIN(timestamp) FROM weather.forecasts WHERE timestamp > CURRENT_TIMESTAMP());");

---

# 8. Stabilità: watchdog e autorecovery

Il firmware include:

✅ watchdog software  
✅ watchdog hardware ESP32  
✅ controllo consistenza sensori  
✅ riavvio forzato se task di rete blocca  
✅ timeout WiFi  
✅ ripartenza automatica dopo blackout  

---

# 9. Logging e debug

La seriale emette con diversi livelli di verbosità configurabili:

- valori dei sensori  
- stato rete  
- aggiornamenti display  
- configurazioni ricevute  
- errore lettura sensori  
- fallback su ultimo valore buono  
- timestamp locali via NTP

---

# 10. Possibili estensioni future

- aggiornamenti OTA over WiFi  
- supporto MQTT opzionale  
- integrazione con Home Assistant  
- smoothing dinamico qualità aria  
- interfaccia dark/light mode  
- pannello impostazioni locali  
- auto‑apprendimento delle abitudini di utilizzo  

---

