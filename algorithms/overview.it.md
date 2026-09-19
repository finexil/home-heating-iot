Panoramica degli algoritmi + architettura decisionale generale.

# Overview degli Algoritmi – Home Heating IoT

Gli algoritmi centrali eseguiti sul Raspberry Pi rappresentano la parte "intelligente"
del sistema. Si occupano di:

✅ analizzare la temperatura interna/esterna  
✅ monitorare la produzione fotovoltaica  
✅ coordinare stufa, caldaia e pompe  
✅ applicare Extra Heating in modo sicuro  
✅ evitare consumi inutili  
✅ mantenere comfort costante  
✅ garantire coerenza tra tutti i dispositivi  

Gli ESP8266/ESP32 sono deliberatamente semplici: raccolgono e mostrano dati.
Le decisioni chiave sono prese **dal server centrale**, dove risiedono più potenza
di calcolo e accesso allo storico del database.

---

# 1. Componenti dell'ecosistema algoritmico

Gli algoritmi principali sono:

- **Extra Heating** → sfrutta l'energia solare in eccesso  
- **Logica Termostufa** → priorità termica rispetto alla caldaia  
- **Coordinamento Caldaia/Pompa**  
- **Monitor energia fotovoltaica** via API Solaredge  
- **Analisi storica temperatura interna**  
- **Controlli di sicurezza multipli**

Ogni algoritmo usa intensivamente:

- tabelle *NOW* (`termostat_temp_now`, `external_temp_hum_now`, `termostat_warming_request`)
- log storici (`heating_state`, `termostat_temp_hum`, `powerDetails`)
- trigger che mantengono aggiornati gli stati

---

# 2. Architettura dati

Gli algoritmi leggono principalmente:

| Tabella | Uso |
|--------|-----|
| `termostat_temp_now` | temperatura attuale stanza |
| `termostat_warming_request` | stato richiesto dai termostati |
| `zone_status` | quali zone sono realmente attive |
| `heating_state` | log caldaia/stufa/pompa |
| `external_temp_hum_now` | meteo locale |
| `solar_now` (se presente) | produzione FV |

---

# 3. Flusso decisionale generale

Diagramma ad alta sintesi:

            ┌───────────────────────────────┐
            │   Lettura stato termostati    │
            └───────────────┬───────────────┘
                            ▼
               ┌──────────────────────────┐
               │ Lettura produzione FV    │
               └────────────┬─────────────┘
                            ▼
         ┌────────────────────────────────────────┐
         │  Logica Extra Heating (fascia oraria)  │
         └─────────────────┬──────────────────────┘
                           ▼
              ┌──────────────────────────┐
              │ Stato Termostufa         │
              └────────────┬─────────────┘
                           ▼
              ┌──────────────────────────┐
              │ Logica Caldaia + Pompa   │
              └────────────┬─────────────┘
                           ▼
              ┌──────────────────────────┐
              │ Aggiornamenti DB         │
              └──────────────────────────┘


---

# 4. Frequenze di aggiornamento

- Extra Heating → ogni **5 minuti**
- Termostufa → a ogni variazione significativa
- Caldaia/Pompa → ogni **90 secondi**
- Temperature NOW → aggiornate dai trigger al volo
- Meteo → ogni **10 minuti**
- API Solaredge → ogni **5 minuti**

---

# 5. Sicurezza e coerenza

Tutto il sistema applica:

✅ hysteresis termiche  
✅ limiti temperatura locali (22°C per extra heating)  
✅ blocco accensioni fuori orario  
✅ blocco caldaia durante stufa attiva  
✅ riavvio MCU automatico in caso di perdita connessione  
✅ controlli incrociati su DB per evitare stati incoerenti  

---

# 6. File specifici nella cartella algorithms/

| File | Contenuto |
|------|-----------|
| `extra_heating.md` | Logica completa dell’Extra Heating |
| `stove_logic.md` | Logica termostufa lato server |
| `adaptive_logic.md` | Analisi e adattamento configurazioni (opzionale) |
| `diagrams.md` | Diagrammi di flusso dettagliati |

---

# 7. Obiettivo

Raccogliere tutta la logica ad alto livello che governa:

- comfort  
- efficienza  
- coordinamento fonti termiche  
- sfruttamento del fotovoltaico  
- riduzione consumi  



