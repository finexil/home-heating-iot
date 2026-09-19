# Diagrammi di Flusso – Algoritmi Home Heating IoT

Questo documento raccoglie i diagrammi di flusso principali degli algoritmi
che governano il sistema Home Heating IoT.

I diagrammi sono volutamente **ad alto livello**, per mostrare il comportamento
logico del sistema senza entrare nel dettaglio implementativo del codice.

---

# 1. Diagramma generale del sistema
                                ┌───────────────┐
                                │  Termostati   │
                                │ ESP8266/ESP32 │
                                └───────┬───────┘
                                        │        
                                        │ INSERT / UPDATE        
                                        ▼        
                           ┌──────────────────────────┐
                           │        Database          │
                           │  (MariaDB + Trigger)     │
                           └───┬──────────────────┬───┘
                               │                  │        
                               │                  │        
                               ▼                  ▼        
                        ┌─────────────┐  ┌──────────────────┐
                        │  Attuatori  │  │ Algoritmi Server │
                        │ Elettroval. │  │  (Raspberry Pi)  │
                        └──────┬──────┘  └────────┬─────────┘
                               │                  │
                               ▼                  ▼
                        ┌─────────────┐   ┌───────────────┐
                        │   Pompa     │   │ Extra Heating │
                        │ + Caldaia   │   │ Termostufa    │
                        └─────────────┘   └───────────────┘

---

# 2. Diagramma algoritmo Extra Heating

L'algoritmo di Extra Heating segue una schedulazione giornaliera che si attiva alle 09:00 tutti i giorni nel periodo Ottobre-Marzo
                          
                          ┌────────────────────────────┐
                          │ Inizio ciclo (ogni 15 min) │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐   
                          │ Fascia oraria valida?      │   
                          │ (es. 9:30–16:30)           │  ─── NO ──┐ 
                          └──────────────┬─────────────┘           ▼
                                         │ SI             ┌──────────────────────┐
                                         ▼                │  Extra Heating OFF   │                                         
                          ┌────────────────────────────┐  └──────────────────────┘
                          │   Leggi produzione FV      │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │  Leggi assorbimento casa   │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │    Calcola surplus FV      │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Batteria in surplus?       │───── NO ───┐
                          └──────────────┬─────────────┘            │
                                         │ SI                       │
                                         ▼                          ▼
                          ┌────────────────────────────┐  ┌──────────────────────┐
                          │ Extra Heating ON           │  │ Extra Heating OFF    │
                          │ Forza richiesta zone       │  └──────────────────────┘
                          └────────────────────────────┘                       
                                                     

---

# 3. Diagramma logica Termostufa (server)

                          ┌────────────────────────────┐
                          │ Ricezione stato stufa      │
                          │ (da controller termostufa) │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ stove_on = 1 ?             │────── NO ────┐
                          └──────────────┬─────────────┘              │
                                         │ SI                         │             
                                         ▼                            ▼
                          ┌────────────────────────────┐ ┌───────────────────────────┐
                          │ f1 di termostatSetup = 1   │ │ Logica normale            │
                          └──────────────┬─────────────┘ │  f1 di termostatSetup = 0 │
                                         |               └───────────────────────────┘
                                         |                             |
                                         ▼                             ▼
                                      ┌───────────────────────────────────┐
                                      │ Gestione riscaldamento tramite    │
                                      │ termostati                        │
                                      └───────────────────────────────────┘

---

# 4. Diagramma logica Caldaia + Pompa

                          ┌────────────────────────────┐
                          │ Ciclo continuo ogni 90s    │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Leggi tabella zone_status  │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Almeno una zona ON?        │──── NO ───┐
                          └──────────────┬─────────────┘           │
                                         │ SI                      ▼
                                         ▼               ┌──────────────────────┐
                          ┌────────────────────────────┐ │ Spegni caldaia       │
                          │ Pompa ON                   │ │ Ritardo → pompa OFF  │
                          └──────────────┬─────────────┘ └──────────────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Ritardo sicurezza          │
                          │ (~30 secondi)              │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Caldaia ON                 │
                          └────────────────────────────┘

---

# 5. Diagramma interazione Termostati / Zone

                          ┌────────────────────────────┐
                          │ Termostato locale          │
                          │ (config + forzature)       │
                          └──────────────┬─────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Temperatura < soglia?      │───────── NO ────────┐
                          └──────────────┬─────────────┘                     │
                                         │ SI                                ▼
                                         ▼                         ┌──────────────────────┐
                          ┌─────────────────────────────────────┐  │ Zona NON attiva      │
                          │ Temperatura < Configurazione Oraria │  └──────────────────────┘
                          |              OR                     |
                          | flag f0 Extra Heating               |
                          |              OR                     |
                          | flag f1 Termostufa                  |
                          └──────────────┬──────────────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Zona attiva                │
                          | Tabella heating_status = 1 |
                          └────────────────────────────┘

---

# 6. Considerazioni finali

Questi diagrammi mostrano:

✅ separazione netta tra sensori, decisioni e attuazione  
✅ assenza di dipendenze dirette tra MCU  
✅ logica deterministica e leggibile  
✅ priorità energetiche chiare  
✅ facilità di estensione futura  

I diagrammi completano la documentazione algoritmica
e forniscono una visione immediata del funzionamento globale del sistema.
