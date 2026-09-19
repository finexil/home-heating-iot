# Introduzione tecnica al progetto Home Heating IoT

Questa sezione fornisce una panoramica tecnica dell’architettura del sistema IoT 
per la gestione del riscaldamento domestico realizzato con ESP8266, ESP32 e Raspberry Pi.

La documentazione qui raccolta è complementare all’introduzione generale presente 
nel README principale e ha lo scopo di fornire una base tecnica per comprendere 
gli articoli successivi più dettagliati.

---

## 🧩 Architettura generale del sistema

Il sistema si basa su tre livelli principali:

### 1. Livello di sensori e interfaccia (Termostati)
- ESP8266 con display 1.8"
- ESP32 con display 3.5" touch
- Sensori: temperatura, umidità, CO₂, TVOC
- Funzioni: grafici 24h, configurazioni orarie, icone dinamiche, previsioni meteo

I termostati non attivano direttamente le elettrovalvole: inviano richieste allo strato centrale.

---

### 2. Livello di attuazione (Controller locali)
- ESP8266 Lolin D1 mini Lite
- Pilotaggio elettrovalvole 230V con relè e transistor
- Timeout per sicurezza idraulica
- Report stato al database centrale

Questa parte gestisce fisicamente la distribuzione dell'acqua calda nelle varie zone.

---

### 3. Livello centrale (Raspberry Pi 2B)
- MariaDB per la gestione dello stato di tutto il sistema
- Algoritmi: Extra Heating, logiche adattative, sincronizzazione
- Web server locale con dashboard per monitoraggio
- Servizi per previsioni meteo e analisi dati

È il “cervello” che coordina termostati, attuatori e generazione del calore.

---

## 🔥 Logiche di riscaldamento

I termostati inviano due indicatori:
- **WARM_STATE** → logica oraria della zona
- **WARM_FORCE** → forzatura manuale ON/OFF

La centrale valuta queste informazioni insieme allo stato:
- del fotovoltaico (per Extra Heating)
- della termostufa
- della caldaia/pompa di calore
- delle temperature interne/esterne

---

## 🌞 Modalità Extra Heating

Quando il sistema rileva disponibilità di energia solare:
- forza i termostati in modalità "riscaldamento extra heating" e
  attiva il riscaldamento in tutte le zone (fiamma gialla)
- arresta automaticamente la zona che supera i **22°C**
- riduce il ricorso alla rete elettrica

---

## 🔥 Integrazione della termostufa

La termostufa, quando attiva:
- fornisce calore a tutto l’impianto
- forza i termostati in modalità "riscaldamento da stufa"
- è rappresentata da un’icona dedicata

La logica della pompa di ricircolo evita shock termici e oscillazioni.

---

## 📡 Comunicazione interna

Tutti i dispositivi comunicano con il Raspberry Pi esclusivamente tramite:
- Wi-Fi locale
- interrogazioni SQL a tabelle dedicate
- trigger per aggiornamenti immediati
- tabelle “now” ottimizzate per il polling rapido degli ESP

Non sono utilizzati servizi cloud.

---

## 📑 Documenti tecnici successivi

Nelle prossime sezioni troverai:

- configurazione hardware di ogni MCU  
- schema delle tabelle del database  
- algoritmi di controllo (Extra Heating, adattativo, sequenze)  
- diagrammi logici del flusso dei dati  
- firmware e struttura del codice  

Questa pagina rappresenta la base tecnica generale da cui partire.
