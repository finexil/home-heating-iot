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
  - **MQ-135**: qualità aria (TVOC/CO₂ stimato)
  - eventuale sensore luminosità per la retroilluminazione
- Alimentazione: 5V → regolazione a 3.3V
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
| MISO     | 19        |
| SCK      | 18        |
| LED      | 15        |

### Touch (XPT2046)

| Funzione | ESP32 Pin |
|----------|-----------|
| T_CS     | 14        |
| T_IRQ    | 27        |

### Sensori

| Sensore | Pin |
|--------|------|
| BME280 (I2C) | SDA=21, SCL=22 |
| MQ-135 | ADC PIN (es: 34) |

---

# 4. FreeRTOS – struttura dei task

Il firmware è strutturato su più task paralleli:

## ✅ 4.1 task_sensori (alta priorità)
- legge temperatura, umidità, pressione  
- legge qualità aria  
- applica filtri e stabilizzazione valori  
- aggiorna variabili condivise thread-safe  

Periodo: **500 ms**

---

## ✅ 4.2 task_display (priorità media)
Aggiorna:

- temperatura corrente  
- icone (rossa/gialla/stufa)  
- grafico miniaturizzato 24h  
- stato WiFi  
- previsioni meteo a 6 ore  
- pagina attiva

Periodo: **200–300 ms**

---

## ✅ 4.3 task_touch (priorità media)
Gestisce:

- tocchi sul display  
- cambio pagina  
- attivazione schermate speciali  
- forzature riscaldamento  
- regolazione retroilluminazione in base al tempo o touch

---

## ✅ 4.4 task_net (bassa priorità)
Invia i dati al server:

- temperatura media ogni 10 minuti  
- umidità  
- qualità aria  
- stato riscaldamento  
- lettura configurazione oraria

Esegue HTTP e parsing JSON con priorità bassa per non disturbare UI e sensori.

Periodo: **10 minuti / on-demand**

---

## ✅ 4.5 task_logiche (priorità media)
Funzioni:

- valuta WARM_STATE  
- applica WARM_FORCE  
- aggiorna icone a seconda di:  
  - extra heating  
  - termostufa  
  - configurazione oraria  
- implementa euristiche locali (es: smoothing della temperatura)

Periodo: **1 secondo**

---

# 5. Interfaccia Grafica (UI)

### ✅ Schermata principale
- Temperatura grande  
- Umidità  
- Qualità aria (barra CO₂/TVOC)  
- Icona riscaldamento (rossa/gialla/stufa)  
- Orario locale  
- Stato WiFi

### ✅ Schermata grafico 24h
- grafico storico (preparato sul server e inviato come array oppure richiesto via HTTP)  
- minima/massima  
- trend

### ✅ Schermata info ambiente
- pressione atmosferica  
- indice qualità aria  
- intensità segnale WiFi  
- uptime del termostato

### ✅ Previsioni meteo integrate (6 ore)
Viste solo nella versione ESP32:

- icona meteo  
- temperatura prevista  
- percentuale di precipitazioni  
- condizioni (sole/velato/pioggia)

Le previsioni provengono dal Raspberry Pi.

---

# 6. Logiche di riscaldamento

Il termostato ESP32 mostra la fonte del calore in modo dinamico:

| Icona | Significato |
|-------|-------------|
| 🔥 Rossa | Richiesta dalla configurazione oraria |
| 🔶 Gialla | Extra Heating (fotovoltaico) |
| 🪵 Stufa | Riscaldamento da termostufa |

Le logiche:

- **Extra Heating** e **Termostufa** forzano tutte le zone in riscaldamento  
- limite automatico a **22°C** per evitare surriscaldamento  
- il termostato mostra la modalità senza confusione

---

# 7. Comunicazione col server

## ✅ Invio temperatura, umidità, qualità aria

/update_temp.php?room=p1_soggiorno&temp=20.9&hum=47&co2=540

## ✅ Invio stato riscaldamento

/update_state.php?room=p1_soggiorno&state=1

## ✅ Ricezione configurazione oraria

/get_config.php?room=p1_soggiorno

## ✅ Ricezione previsioni meteo

/get_forecast.php

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

La seriale emette:

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

