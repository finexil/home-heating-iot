# Logica Termostufa – Algoritmo lato Server

Questo documento descrive la **logica di gestione della termostufa**
implementata sul server centrale (Raspberry Pi), che coordina
il comportamento di caldaia, pompa di ricircolo, termostati e attuatori.

La termostufa è trattata come **fonte di calore prioritaria** rispetto
alla caldaia elettrica e all’Extra Heating.

---

# 1. Ruolo della termostufa nel sistema

Nel sistema Home Heating IoT, la termostufa:

✅ non comanda direttamente le zone  
✅ non pilota attuatori o caldaia  
✅ fornisce **informazione energetica** al sistema  
✅ viene monitorata dal controller dedicato  
✅ influenza le decisioni degli algoritmi centrali  

Il suo stato è propagato tramite il database.

---

# 2. Origine del dato “termostufa attiva”

Il controller termostufa:

- legge la temperatura dell’acqua nel serbatoio caldaia
- applica hysteresis ON > 60°C, OFF < 55°C
- aggiorna il campo `termostufa` nella tabella `zone_status` e il flag `f1` nella tabella `termostatSetup`

Il server **non misura direttamente** la termostufa,
ma si fida dello stato validato dal controller dedicato.

---

# 3. Tabelle coinvolte

La logica termostufa utilizza principalmente:

| Tabella | Uso |
|------|-----|
| `termostatSetup` | configurazione oraria dei Termostati |
| `heating_state` | Stato stufa, caldaia, pompa |
| `zone_status` | Zone effettivamente aperte |
| `termostat_warming_request` | Richieste aggregate dei termostati |
| `termostat_temp_now` | Temperature locali |

---

# 4. Priorità energetiche

La gerarchia delle fonti di calore è:

1. <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/53d6b121-8ec9-4bc0-9891-884fe79045de" /> **Termostufa**
2. <img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/6389740f-86ea-4227-b1d1-031ebf1e72d3" /> **Extra Heating (fotovoltaico)**
3. <img width="40" height="39" alt="image" src="https://github.com/user-attachments/assets/d97f6b0c-e857-4e03-8112-6adf18e54451" />  **Caldaia elettrica**

Questa priorità è **rigida** e applicata in ogni ciclo decisionale.

---

# 5. Logica di attivazione termostufa

## ✅ 5.1 Termostufa attiva (`stove_on = 1`)

Quando la termostufa è attiva:

✅ la caldaia elettrica è **forzata OFF** dal sistema di regolazione interno che si accorge avere una fonte di `energia` esterna 
✅ l’Extra Heating è **ignorato** fintanto che la termostufa è attiva
✅ la pompa di ricircolo della termostufa è attiva, mentre quella della caldaia è attiva in base allo stato delle zone della casa  
✅ i termostati visualizzano l’icona **STUFA**  
✅ le zone vengono scaldate solo se la temperatura locale registrata è sotto soglia (22°C)

La termostufa **non forza indiscriminatamente** tutte le zone:
sono sempre i termostati a decidere se la stanza accetta calore.

## ✅ 5.2 Termostufa spenta (`stove_on = 0`)

Quando la termostufa è spenta:

- il sistema ritorna alla logica normale
- Extra Heating può attivarsi
- la caldaia elettrica è disponibile alle logiche dei termostati e dell'eventuale Extra Heating
- le zone seguono la configurazione oraria della tabella termostatSetup

---

# 6. Interazione con i termostati

Quando `stove_on = 1`:

- i termostati **non cambiano la propria logica interna**
- continuano a valutare:
  - configurazione oraria tra cui i flag f0 (Extra Heating) e f1 (Temostufa)
  - forzature locali
  - limite massimo di temperatura
- mostrano però l’icona **<img width="39" height="39" alt="image" src="https://github.com/user-attachments/assets/53d6b121-8ec9-4bc0-9891-884fe79045de" /> STUFA** per indicare la fonte del calore qualora la temperatura locale sia sotto soglia massima

Questo evita confusione per l’utente finale.

---

# 7. Interazione con la pompa di ricircolo

La pompa di ricircolo caldaia:

✅ può essere attiva anche senza la pompa di calore  
✅ viene accesa se almeno una zona è aperta  
✅ segue i ritardi di sicurezza del controller caldaia  

In modalità stufa:

- la pompa serve solo a distribuire il calore
- la pompa di calore resta sempre spenta

---

# 8. Disabilitazione Extra Heating

Se `stove_on = 1`:
extra_heating_on = X, qualsiasi sia il suo valore, la stufa prende il sopravvento 

Motivazioni:

- la stufa fornisce già energia termica
- evitare sovrapposizione di fonti
- evitare sprechi di energia elettrica
- evitare cicli inutili della caldaia

Extra Heating rientra in gioco automaticamente quando la stufa si spegne.

---

# 9. Sicurezze e coerenza

La logica del controller e del server applica:

✅ hysteresis lato stufa (applicata dal controller)  
✅ controllo incrociato con `zone_status` (applicata dal controller)
✅ verifica che la pompa non resti accesa inutilmente (applicata dal controller)  
✅ logging di ogni transizione STUFA ON/OFF (applicata dal controller e dal server)
✅ fallback in caso di stato incoerente (applicata dal controller e dal server)

In caso di valori non validi:

→ comportamento conservativo  
→ spegnimento caldaia  
→ log evento  

---

# 10. Logging e analisi

Ogni evento rilevante viene tracciato in `heating_state` con:

- timestamp  
- stato stufa  
- stato caldaia  
- stato pompa  
- motivo della decisione  

Questo storico è utilizzato:

✅ per debug  
✅ per analisi stagionali  
✅ per migliorare gli algoritmi futuri  

---

# 11. Vantaggi di questa architettura

✅ separazione netta tra sensori e decisioni  
✅ nessuna dipendenza diretta tra MCU  
✅ priorità energetiche chiare  
✅ comportamento deterministico  
✅ facilità di manutenzione  
✅ possibilità di estensione futura  

---

# 12. Collegamenti ad altri documenti

- `extra_heating.md` → logica FV  
- `boiler_controller.md` → logica pompa/caldaia  
- `stove_controller.md` → firmware MCU termostufa  
- `overview.md` → architettura algoritmi  
