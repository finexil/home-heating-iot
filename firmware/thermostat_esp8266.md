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
| Display A0/DC | D1 | SPI |
| Display RESET | D0 | Reset logico |
| SPI SCK | D5 | Standard |
| SPI MOSI | D7 | Standard |
| SPI MISO | D6 | *non usato dal display* |
| Sensore DHT22 | D3 | Lettura temperatura/umidità |
| Pulsante PAGE | D4 | Pulldown |
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

L’ESP8266 usa HTTP GET verso endpoint PHP sul Raspberry Pi.

## 5.1 Invia temperatura media (ogni 10 minuti)


/update_temp.php?room=pt_camera&temp=21.4&hum=46

## 5.2 Invia stato riscaldamento


/update_state.php?room=pt_camera&state=1

## 5.3 Riceve configurazione oraria


/get_config.php?room=pt_camera

La logica PHP aggiorna le tabelle:

- `termostat_temp_hum`
- `Warming_state`

I trigger si occupano del resto.

---

# 6. Interfaccia utente

## 6.1 Display 1.8” – pagine disponibili

1. **Schermata principale**
   - Temperatura attuale
   - Umidità
   - Icona fonte calore
2. **Grafico 24 ore**
   - Mini-grafico della temperatura
3. **Schermata info**
   - Indirizzo IP
   - Potenza WiFi
   - Versione firmware

---

## 6.2 Icone di riscaldamento

Le icone sono identiche alla versione ESP32 e indicano la fonte di calore:

| Icona | Descrizione |
|-------|-------------|
| 🔥 (rossa) | richiesta locale (configurazione oraria) |
| 🔶 (gialla) | extra heating da fotovoltaico |
| 🪵 (stufa) | riscaldamento da termostufa |

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
