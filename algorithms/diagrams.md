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
└───────┬─────────┬────────┘
        │         │
        │         │
        ▼         ▼
┌─────────────┐  ┌──────────────────┐
│  Attuatori  │  │ Algoritmi Server │
│ Elettroval. │  │  (Raspberry Pi)  │
└──────┬──────┘  └─────────┬────────┘
       │                   │
       ▼                   ▼
┌─────────────┐     ┌───────────────┐
│   Pompa     │     │ Extra Heating │
│ + Caldaia   │     │ Termostufa    │
└─────────────┘     └───────────────┘

---

# 2. Diagramma algoritmo Extra Heating


┌────────────────────────────┐
│ Inizio ciclo (ogni 5 min)  │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Leggi produzione FV        │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Leggi assorbimento casa    │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Calcola surplus FV         │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Batteria in surplus?       │─── NO ──┐
└──────────────┬─────────────┘         │
│ SI                     │
▼                        ▼
┌────────────────────────────┐   ┌──────────────────────┐
│ Fascia oraria valida?      │   │ Extra Heating OFF    │
│ (es. 9:30–16:30)           │   └──────────────────────┘
└──────────────┬─────────────┘
│ SI
▼
┌────────────────────────────┐
│ Mese abilitato?            │
│ (es. Ott–Mar)              │
└──────────────┬─────────────┘
│ SI
▼
┌────────────────────────────┐
│ Termostufa attiva?         │─── SI ──┐
└──────────────┬─────────────┘         │
│ NO                     ▼
▼                ┌──────────────────────┐
┌────────────────────────────┐ │ Extra Heating OFF    │
│ Almeno una stanza < 22°C?  │ └──────────────────────┘
└──────────────┬─────────────┘
│ SI
▼
┌────────────────────────────┐
│ Extra Heating ON           │
│ Forza richiesta zone       │
└────────────────────────────┘

---

# 3. Diagramma logica Termostufa (server)


┌────────────────────────────┐
│ Ricezione stato stufa      │
│ (da controller termostufa) │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ stove_on = 1 ?             │─── NO ──┐
└──────────────┬─────────────┘         │
│ SI                     ▼
▼                ┌──────────────────────┐
┌────────────────────────────┐ │ Logica normale       │
│ Disabilita Extra Heating   │ │ (caldaia + FV)       │
└──────────────┬─────────────┘ └──────────────────────┘
▼
┌────────────────────────────┐
│ Forza caldaia OFF          │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Gestione pompa in base     │
│ alle zone aperte           │
└────────────────────────────┘

---

# 4. Diagramma logica Caldaia + Pompa


┌────────────────────────────┐
│ Inizio ciclo (ogni 90s)    │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Leggi zone_status          │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Almeno una zona ON?        │─── NO ──┐
└──────────────┬─────────────┘         │
│ SI                     ▼
▼                ┌──────────────────────┐
┌────────────────────────────┐ │ Spegni caldaia       │
│ Leggi stove_on              │ │ Ritardo → pompa OFF │
└──────────────┬─────────────┘ └──────────────────────┘
▼
┌────────────────────────────┐
│ stove_on = 1 ?             │─── SI ──┐
└──────────────┬─────────────┘         │
│ NO                     ▼
▼                ┌──────────────────────┐
┌────────────────────────────┐ │ Pompa ON             │
│ Pompa ON                   │ │ Caldaia OFF          │
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
│ Temperatura < soglia?      │─── NO ──┐
└──────────────┬─────────────┘         │
│ SI                     ▼
▼                ┌──────────────────────┐
┌────────────────────────────┐ │ Zona NON attiva      │
│ Accetta richiesta          │ └──────────────────────┘
│ (anche Extra Heating)      │
└──────────────┬─────────────┘
▼
┌────────────────────────────┐
│ Zona attiva                │
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
