# Termostato ESP8266 con display 1.8" – Dettagli del Firmware

Questo documento descrive la logica, la struttura e il funzionamento del firmware
del termostato basato su **ESP8266 (modulo ESP-12E)** con display TFT da **1.8"**.

Viene utilizzato nelle camere e nei bagni, come versione “compatta” del termostato principale.

---

# 1. Architettura generale

Il firmware ha questi obiettivi:

✅ leggere periodicamente temperatura/umidità  
✅ applicare la configurazione oraria della zona  
✅ gestire pulsanti fisici: pagina & forzatura  
✅ visualizzare grafico e icone di stato  
✅ inviare al DB la media delle letture ogni 10 minuti  
✅ segnalare lo stato (ON/OFF/FORCE) quando cambia  

Il dispositivo è progettato per essere **robusto**, **reattivo** e **semplice da aggiornare**.

---

# 2. Hardware e connessioni

## 2.1 Componenti principali

- MCU: **ESP8266 ESP-12E**
- Display: **TFT 1.8” SPI** (ST7735)
- Sensore temperatura/umidità:
  - DHT22 (tipico)
- Due pulsanti fisici:
  - **PAGE_SELECT**
  - **WARM_FORCE**
- Alimentazione: **5V → AMS1117 3.3V**

---

## 2.2 Pinout consigliato

| Funzione | Pin ESP8266 | Note |
|---------|-------------|------|
| Display CS  | D2 | SPI |
| Display A0/DC | D4 | SPI |
| Display RESET | D3 | Reset logico |
| SPI SCK | D5 | Standard |
| SPI MOSI | D7 | Standard |
| SPI MISO | N.C. | *non usato dal display* |
| Sensore DHT22 | D1 | Lettura temperatura/umidità |
| Pulsante PAGE | D6 | Pulldown |
| Pulsante FORCE | D8 | Pulldown |
| WiFi | Interno | Sempre attivo |

---

# 3. Ciclo operativo del termostato

Il termostato usa un ciclo semplice ma molto efficace:

LOOP PRINCIPALE (ogni 5 secondi)
→ Leggi sensore temperatura/umidità
→ Aggiorna media mobile per 10 minuti
→ Aggiorna display (icona, temperatura, grafico)
→ Verifica stato WARM_STATE vs WARM_FORCE
→ Se cambia stato → invia nuovo stato al DB
→ Ogni 10 minuti → invia temperatura media

Questo ciclo garantisce:

✅ reattività  
✅ consumi ridotti  
✅ stabilità del WiFi  
✅ trafﬁco minimo sul server  

---

# 4. Logica WARM_STATE e WARM_FORCE

### ✅ **WARM_STATE**  
Deriva dalla **configurazione oraria** ottenuta dal Raspberry Pi:

- 0 → non riscaldare
- 1 → riscaldare

Viene scaricata dal server tramite HTTP ogni pochi minuti.

---

### ✅ **WARM_FORCE**
Gestito dal pulsante fisico:

- **0 = NORMAL** (segue WARM_STATE)
- **1 = FORCE ON**
- **2 = FORCE OFF**

Il valore finale inviato al DB è:


richiesta =
FORCE OFF → 0
FORCE ON  → 1
NORMAL    → WARM_STATE

---

# 5. Comunicazione con il database

L’ESP8266 usa la libreria ![MySQL_MariaDB_Generic](https://github.com/khoih-prog/MySQL_MariaDB_Generic) nelle funzioni di INSERT, UPDATE e SELECT.

## 5.1 Invia temperatura media (ogni 10 minuti)

```sql
"INSERT INTO temperature.powerDetails
(room, Temperature, Humidity, Pressure)
VALUES ('%s', %.3f, %.3f, %.3f);", room[my_room_number], tf, hf, 0.0
```

## 5.2 Invia stato riscaldamento

```sql
"INSERT INTO temperature.Warming_state
(room, FLG_ON, FLG_FORCE)
VALUES ('%s', %d, %d)", room[my_room_number], heating_status, heating_mode
```

## 5.3 Riceve configurazione oraria

```sql
"SELECT * FROM temperature.TermostatSetup
WHERE room='%s';", room[my_room_number]
```

Questa tabella contiene l'impostazione oraria e i due flag per le forzature da Extra Heating e Termostufa:
- Extra Heating F0
- Termostufa F1

---

# 6. Interfaccia utente

## 6.1 Display 1.8” – pagine disponibili

1. **Schermata principale**
   - Temperatura attuale
   - Umidità
   - Temperatura esterna
   - Temperatura oraria impostata
   - Icona fonte calore
   - Icone potenza segnale Wifi
2. **Grafico 24 ore**
   - Mini-grafico della temperatura e dell'umidità
   - Mini grafico con l'impostazione oraria delle temperature e andamento della temperatura della zona
3. **Schermata info**
   - Indirizzo IP
   - Stato della connessione al DB
   - Versione firmware
   - timestamp ultimo boot

---

## 6.2 Icone di riscaldamento

Le icone sono identiche alla versione ESP32 e indicano la fonte di calore:

| Icona | Descrizione |
|-------|-------------|
| <img width="40" height="39" alt="image" src="https://github.com/user-attachments/assets/d97f6b0c-e857-4e03-8112-6adf18e54451" /> | richiesta locale (configurazione oraria) |
| <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/6389740f-86ea-4227-b1d1-031ebf1e72d3" /> | extra heating da fotovoltaico |
| <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/53d6b121-8ec9-4bc0-9891-884fe79045de" /> | riscaldamento da termostufa |

L’icona è decisa via DB leggendo un parametro della tabella `heating_state`.

---

# 7. Gestione pulsanti

### ▶️ **Pulsante PAGE_SELECT**
Cicla tra le schermate del display.  
Implementazione semplice con “debounce” software.

### 🔥 **Pulsante WARM_FORCE**
Passa tra:
- NORMAL → FORCE ON → FORCE OFF

Ogni cambiamento genera **invio immediato** al DB.

---

# 8. Stabilità, sicurezza e watchdog

Il firmware prevede:

✅ timeout WiFi  
✅ riavvio forzato se necessario  
✅ watchdog hardware dell’ESP  
✅ riconnessione automatica  
✅ controllo validità del sensore DHT22  
✅ fallback: se DHT fallisce → invia ultimo valore noto  

---

# 9. Logging

Il log viene inviato via seriale, includendo:

- temperatura/umidità rilevate  
- stato configurazione  
- stato forzatura  
- esito invio dati al server  
- reconnect WiFi  

Questo facilita debug e manutenzione.

---

# 10. Note per versioni future

Possibili estensioni:

- supporto a MQTT (opzionale)
- sincronizzazione NTP più rapida
- supporto a OTA update (ESP8266 permette aggiornamenti WiFi)
- integrazione con sensori CO₂ “leggeri” (tipo MH-Z19)

---
