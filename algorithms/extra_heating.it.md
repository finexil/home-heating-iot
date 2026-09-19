Documento tecnico completo dell’Extra Heating basato su fotovoltaico, batteria, produzione/consumi, limiti dei termostati.

# Algoritmo Extra Heating – Sfruttamento Intelligente del Fotovoltaico

L’algoritmo **Extra Heating** permette al sistema di utilizzare l’energia elettrica
in eccesso prodotta dal fotovoltaico per riscaldare la casa **quando conviene**,
riducendo consumi e migliorando il comfort.

È uno degli elementi più innovativi del progetto, perché combina:

✅ dati energetici in tempo reale  
✅ logiche termiche basate sui termostati  
✅ limiti di sicurezza (max 22°C per stanza)  
✅ comportamento dell’impianto reale (inerzia, ritardi, cicli)  
✅ gestione stagionale e per fasce orarie

---

# 1. Obiettivi dell’algoritmo

L’Extra Heating è stato progettato per:

- usare energia solare **in surplus** senza costi aggiuntivi  
- pre-riscaldare la casa **quando ha senso farlo**  
- rispettare la temperatura massima di comfort  
- intervenire solo in fasce orarie favorevoli  
- garantirsi di non sprecare energia quando la casa è già calda  
- evitare continui ON/OFF improduttivi  

---

# 2. Condizioni di attivazione

L’Extra Heating si attiva **solo se tutte le condizioni seguenti sono vere**:

✅ produzione FV elevata (soglia configurabile)  
✅ batteria domestica carica o in surplus  
✅ assorbimento casa < produzione (surplus reale)  
✅ fascia oraria favorevole (es. 9:30 → 16:30)  
✅ mese dell’anno compreso nel periodo utile (tipicamente ottobre → marzo)  
✅ almeno una stanza sotto **22°C**  

Se una di queste condizioni non è soddisfatta → extra heating **non si attiva**.

---

# 3. Dati utilizzati dall’algoritmo

L’algoritmo legge periodicamente:

## ✅ 3.1 Dati energia (via API Solaredge o equivalente)
- produzione FV in tempo reale  
- stato della batteria (carica / scarica)  
- potenza assorbita dalla casa  
- surplus istantaneo (produzione – consumi)

## ✅ 3.2 Dati termici
- temperatura interna da `termostat_temp_now`  
- temperatura esterna da `external_temp_hum_now`  
- stato richiesto dai termostati  
- stato reale delle zone (`zone_status`)

## ✅ 3.3 Stato impianto
- eventuale termostufa attiva  
- stato pompa/caldaia (`heating_state`)  
- orario reale  
- mese dell’anno  

---

# 4. Attivazione Extra Heating

Se tutte le condizioni sono vere, il sistema:

### ✅ 4.1 Forza tutti i termostati in richiesta riscaldamento  

Imposta nelle tabelle:
termostatSetup viene forzato f0 a 1 per tutte le zone
i termostati leggono il flag f0 e scrivono un nuovo record in termostat_warming_request con lo stato a 1 


### ✅ 4.2 Ma ogni stanza può bloccare la richiesta  
Se la stanza supera **22°C**, il termostato:

- NON segue il forcing  
- non attiva la zona  
- continua ad aggiornare `powerDetails`

Questa è una sicurezza fondamentale.

### ✅ 4.3 Accensione impianto  
I controller attuatori leggo la tabella termostat_warming_request e iniziano ad aprire gli attuatori, quindi portano a 1 il valore della zona nella tabella zone_status.  
il controller della caldaia legge la tabella zone_status e attiva il riscaldamento (pompa + caldaia con ritardo) se almeno una zona è impostata ad on (valore 1).

---

# 5. Disattivazione Extra Heating

Extra Heating si disattiva quando:

- il surplus FV sparisce  
- si esce dalla fascia oraria  
- si entra in un mese non ammesso  
- tutte le stanze raggiungono 22°C  
- la termostufa si attiva  

L’algoritmo imposta quindi:

tabella termostatSetup imposta il flag f0 a 0 per tutte le zone

E ripristina la logica normale.

---

# 6. Ciclo operativo

Ogni 5 minuti:


LOOP EXTRA HEATING
→ Leggi produzione FV
→ Leggi assorbimento casa
→ Calcola surplus FV
→ Leggi stato batteria
→ Leggi temperature interne
→ Verifica fascia oraria
→ Verifica mese
→ Se tutte le condizioni vere:
→ imposta extra_heating_on=1
→ forza riscaldamento per tutte le zone, i termostati non attiveranno il riscaldamento se la temperatura della zona non supera i 22°C
Altrimenti:
→ imposta extra_heating_on=0

---

# 7. Principio dei 22°C (comfort + sicurezza)

Extra Heating **non serve a surriscaldare la casa**, ma a:

- stabilizzare temperatura  
- sfruttare energia gratuita  
- compensare inerzia del radiante  

Il limite 22°C:

✅ evita discomfort  
✅ impedisce sprechi  
✅ mantiene la logica semplice  
✅ impedisce alla caldaia di partire inutilmente  

---

# 8. Interazioni con gli altri componenti

## ✅ 8.1 Termostufa  
Se la termostufa è attiva (stove_on = 1):

- Extra Heating rimane attivo ma la logica di priorità privilegia la termostufa
- le zone sono gestite dalla termostufa
- il sistema evita conflitti energetici

## ✅ 8.2 Controller Caldaia  
Segue la stessa logica di attivazione delle zone,  
ma:

- accende la caldaia solo se davvero necessario  
- evita avvii brevi o continui

## ✅ 8.3 Attuatori  
Extra Heating impone la richiesta, ma:

- gli attuatori aprono solo le zone che restano sotto 22°C  
- questo impedisce cicli inutili su zone già calde  

---

# 9. Logging

Ogni attivazione o disattivazione viene registrata in:


heating_state.reason = 'EXTRA HEATING ON'
heating_state.reason = 'EXTRA HEATING OFF'

Con:

- orario  
- surplus FV  
- temperatura interna  
- temperatura esterna  
- zone attive  

Questo storico è utile per analisi e ottimizzazioni future.

---

# 10. Estensioni future

- integrazione con previsioni meteo per attivazione pre‑sunrise  
- soglie dinamiche basate su produzione media del giorno  
- machine learning su pattern FV/stanze  
- integrazione con batteria più complessa (SoC dinamico)  
- logica con curva radiante anticipata  
