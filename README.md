# home-heating-iot
Sistema DIY per la gestione del riscaldamento domestico con ESP8266, ESP32 e Raspberry Pi

# Il tuo IoT per il riscaldamento di casa: ESP, Raspberry Pi e algoritmi intelligenti

Gestire in modo intelligente il riscaldamento di una casa non è semplice: significa combinare
**termo-tecnica, elettronica, programmazione, automazione** e una buona dose di pazienza.

Questo progetto nasce dall’esigenza di controllare un impianto complesso:

- riscaldamento a pavimento zonizzato  
- caldaia elettrica  
- pompa di calore  
- termostufa a legna  
- impianto fotovoltaico con accumulo  

Il tutto tramite un sistema completamente **DIY**, basato su:

- **ESP8266**  
- **ESP32**  
- **Raspberry Pi 2B** (unità centrale, DB e web server locale)

Il risultato è un ecosistema IoT che:

- rileva temperatura e umidità per ogni stanza  
- gestisce elettrovalvole, caldaia, pompa di calore e termostufa  
- applica logiche orarie e algoritmi adattivi  
- sfrutta il surplus fotovoltaico  
- memorizza anni di dati per analisi  
- funziona **senza cloud** e completamente in rete locale  

---

## 1. Perché costruire un IoT per il riscaldamento?

Molti sistemi commerciali non gestiscono bene:

- impianti ibridi (pompa di calore + caldaia + termostufa)  
- logiche personalizzate per ogni stanza  
- interazioni con sensori esterni  
- algoritmi di ottimizzazione energetica  

Un sistema DIY consente di definire logiche su misura per la propria casa.

---

## 2. Architettura generale del sistema

L’impianto si basa su tre blocchi principali:

### ✅ Termostati locali (ESP8266 e ESP32)
Per rilevare temperatura/umidità, mostrare grafici, ricevere configurazioni e inviare richieste.

### ✅ Controller di zona e della generazione
- attuatori elettrovalvole  
- controller della caldaia e della pompa  
- controller della termostufa  

### ✅ Unità centrale su Raspberry Pi
- MariaDB  
- web server locale  
- algoritmi di analisi e adattamento  
- gestione previsioni meteo  
- dashboard locale  

---

## 3. I termostati: hardware e logiche

Due modelli:

### 🔹 Termostati con display 1.8" (ESP8266)
Usati in camere e bagni.

### 🔹 Termostati con display 3.5" touch (ESP32)
Usati nelle zone principali / soggiorni.

### Funzioni comuni
- rilevazione temperatura/umidità  
- configurazione oraria  
- grafico delle ultime 24h  
- richiesta di riscaldamento  
- gestione forzature (ON / OFF / AUTO)  
- invio media ogni 10 minuti al DB  

### 🌡️ Sensori aggiuntivi
Nei modelli touch:
- CO₂  
- TVOC  
- Previsioni meteo delle prossime 6 ore

### 🔥 Icone intelligenti
Il display mostra la fonte del riscaldamento:

- **Fiamma rossa** → richiesta del termostato (configurazione oraria)  
- **Fiamma gialla** → Extra Heating (fotovoltaico)  
- **Icona stufa** → riscaldamento da termostufa  

➡️ Extra Heating e Termostufa forzano *tutte le zone* in riscaldamento,  
ma si fermano automaticamente quando la stanza supera **22°C**.

---

## 🔧 Dietro le quinte: FreeRTOS sugli ESP32

Per gestire:

- sensori  
- display TFT  
- WiFi  
- logica locale  

i termostati touch ESP32 utilizzano **FreeRTOS**, con task dedicati:

- lettura sensori (alta priorità)  
- aggiornamento display (media priorità)  
- comunicazione rete (bassa priorità)  
- logiche interne (media)

Risultato:
✅ nessun blocco  
✅ UI fluida  
✅ comportamento deterministico  

---

## 4. Controller delle zone e della caldaia

### Controller elettrovalvole (ESP8266)
- leggono lo stato dal DB  
- aprono/chiudono le valvole  
- rispettano timeout per non stressare la caldaia  
- aggiornano il proprio stato nel DB

### Controller caldaia
Gestisce:

- accensione caldaia  
- attivazione pompa di ricircolo  
- interazione con la termostufa  
- logiche di ritardo (per evitare cicli rapidi ON/OFF)

---

## 5. Raspberry Pi: database + server web locale

Il Raspberry Pi 2B è il cuore del sistema.

### ✅ MariaDB  
Contiene:

- dati dei termostati  
- configurazioni orarie  
- comandi delle zone  
- trigger e tabelle *now* per ottimizzare il polling  
- dati storici (anni di registrazioni)

### ✅ Web server locale  
Non esposto su Internet.  
Permette di visualizzare:

- stato dei termostati  
- grafici storici  
- zone attive  
- previsioni meteo  
- storico dei consumi e temperature  

---

## 6. Algoritmi intelligenti: Extra Heating

Il Raspberry legge:

- produzione fotovoltaico  
- stato batteria  
- consumi domestici  

Quando c’è surplus:

- attiva tutte le zone (fiamma gialla)  
- rispetta il limite dei 22°C  
- massimizza il comfort sfruttando energia “gratuita”

---

## 7. Giornata tipo

1. i termostati leggono T/H ogni 5 secondi  
2. inviano medie ogni 10 minuti  
3. applicano configurazioni orarie  
4. gli attuatori aprono/chiudono valvole  
5. la centrale decide se attivare caldaia / pompa / termostufa  
6. se c’è surplus solare → Extra Heating  
7. se la termostufa scalda → icona stufa  

---

## 8. Sfide tecniche affrontate

- sincronizzazione tra MCU e DB  
- gestione dell’inerzia del pavimento radiante  
- evitare condizioni di gara (race condition)  
- parallelizzazione su ESP32  
- gestione robusta del WiFi  
- controllo intelligente del sistema ibrido caldaia + pompa di calore + termostufa  
- sfruttamento dell’energia fotovoltaica

---

## 9. Conclusioni

Questo progetto dimostra che è possibile realizzare un sistema di riscaldamento:

- **completamente DIY**  
- **altamente personalizzabile**  
- **robusto e affidabile**  
- **ottimizzato per un impianto complesso reale**  
- **senza cloud**, 100% locale  
- **basato su algoritmi intelligenti**

Una base solida per sviluppi futuri come:

- integrazione con Home Assistant  
- machine learning sul comportamento termico della casa  
- automazioni basate su previsioni meteo  
- dashboard avanzate  
- controllo vocale  

---

## 📂 Licenza

Questo progetto è rilasciato sotto licenza **MIT**.  
Vedi il file `LICENSE` per i dettagli

