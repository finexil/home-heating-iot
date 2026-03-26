# Controller Termostufa – Firmware ESP8266

Il controller termostufa è responsabile del rilevamento dello stato della stufa e della
temperatura del circuito acqua collegato allo scambiatore della caldaia.  
Il suo compito è:

✅ segnalare al sistema quando la termostufa è attiva  
✅ coordinarsi con la logica centrale (caldaia + pompe)  
✅ forzare correttamente le zone in “riscaldamento da stufa”  
✅ garantire sicurezza termica e comportamento deterministico  

È un componente semplice nella struttura, ma critico per il comportamento dell’impianto.

---

# 1. Architettura generale

Il controller:

- legge un **sensore di temperatura acqua** (interruttore termico sul serbatoio della caldaia)
- determina se la termostufa sta generando calore utile
- apre una connessione diretta MySQL (via libreria ![MySQL_MariaDB_Generic](https://github.com/khoih-prog/MySQL_MariaDB_Generic))
- aggiorna lo stato `stove_on` nelle tabelle `termostufa` e 'zone_status'
- aggiorna il flag f1 a on della tabella termostatSetup che permette agli altri dispositivi (termostati, attuatori, caldaia) di riconoscere la modalità “STUFA”

L’MCU **NON comanda direttamente nulla nell’impianto** → fornisce solo lo “stato energia”.

---

# 2. Hardware

- MCU: **ESP32 D1 Mini**
- Sensore temperatura acqua:  
  - PT100/NTC con convertitore → ADC dell’ESP 
- ingresso contatto interruttore termico della termostufa
- pilotaggio relè per attivare la pompa di ricircolo dell'acqua verso la caldaia
- Alimentazione filtrata + stabilizzata 3.3V

---

# 3. Sensore e rilevamento stato termico

### ✅ Temperature tipiche nel tuo impianto
- Stufa considerata **attiva** quando:
  - temperatura serbatoio caldaia > **60°C**  e l'interruttore termico si chiude

- Stufa considerata **spenta** quando:
  - temperatura scende sotto ~**55°C** e l'interruttore termico si chiude 

In fase di riscaldamento/spegnimento, nonostante i circa 5°C di isteresi, l'interruttore termico interviene più volte con stati alternati on/off
poichè il ricircolo dell'acqua verso la caldaia influisce spece in fase di avvio sulla temperatura dell'acqua riscaldata dalla termostufa.
Per evitare accensioni e spegnimenti continui della pompa di ricircolo sono introdotti dei loop di timeout all'interno dei quali lo stato della pompa non varia; terminato il timeout, qualora lo stato richiesto sia diverso da quello precedente all'avvio del timeout, la pompa entra nel nuovo stato desiderato. 


### ✅ Mappatura stato

| Condizione | Stato |
|-----------|-------|
| T > 60°C | `stove_on = 1` |
| T < 55°C | `stove_on = 0` |
| T tra 55–60°C | mantiene stato precedente fino all'esaurirsi del timeout definito|

---

# 4. Connessione diretta MySQL

Il firmware usa connessione diretta MariaDB:

Per tutte le MCU basate su ESP8266, la connessione al DB avviene tramite la libreria ![MySQL_MariaDB_Generic](https://github.com/khoih-prog/MySQL_MariaDB_Generic).

Permessi dell’utente DB:

SELECT
INSERT → log termostufa
UPDATE → aggiornamento stato

Nessun altro permesso.

# 5. Query SQL utilizzate
 # 5.1 ✅ Lettura stato zone (per gestione coerenza stati)

```sql
SELECT pt_soggiorno, pt_camera, pt_bagno, p1_soggiorno, p1_camera, p1_bagno, te_terrazzo 
FROM temperature.zone_status WHERE location='ceresole';
```

 # 5.2 ✅ Aggiornamento stato termostufa (tabella dedicata intesa come log)

```sql
INSERT INTO  temperature.termostufa (Stato , Condizione , Temperatura)
VALUE ('## stato termostufa ##', 0/1, 0/1);
```

 # 5.3 ✅ Impostazione stato per termostati

 L'impostazione dello stato richiesto ai termostati avviene impostando il valore 0/1 al flag f1 della tabella termostatSetup per ogni zona (stanza).
 In particolare il flag viene gestito per l'impostazione al valore 1 (termostufa attiva) raggruppando le zone secondo il seguente criterio:
  - bagni
  - camere
  - soggiorno p1
  - soggiorno pt
  - terrazzo (è il nome di un locale chiuso ex fienile)
Questi raggruppamenti sono pensati per attivare le varie zone con tempi diversi così da non stressare troppo il riscaldamento dell'acqua da parte della termostufa.
L'impostazione del valore a 0 (termostufa spenta) avviene in contemporanea su tutte le zone.

# Impostazione flag a 1 per le zone bagni
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%%bagno';
```
# Impostazione flag a 1 per le zone camere
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%%camera';
```

# Impostazione flag a 1 per soggiorno piano primo
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room =  'p1 soggiorno';
```

# Impostazione flag a 1 per soggiorno piano terra
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room =  'pt soggiorno';
```

# Impostazione flag a 1 per terrazzo
```sql
UPDATE temperature.TermostatSetup SET f1=1 WHERE room LIKE '%%terrazzo';
```

# 6. Flusso operativo del firmware
Ogni 40 ms/6 s:

LOOP PRINCIPALE
    → Leggi temperatura acqua
    → Leggi stato_stufa (gestione hysteresis)
    → Se cambio stato è confermato dopo timeout:
         → UPDATE tabella termostufa (campo stove_on)
         → Log variazione

Il delay del loop principale è configurato per cambiare a secnda se la termostufa sia attiva o disattiva, in pratica se la termostufa
è in stato attivo e sono presenti dei cambiamenti di stato, il delay è di 40 ms, mentre è di 6 secondi in situazione di "standby".


# 7. Interazione con il resto dell’impianto
La termostufa non comanda direttamente lo stato delle zone o la caldaia.
È la logica dei termostati sollecitati tramite il flag f1 della tabella termostatSetup a reagire allo stato stove_on.
✅ Se stove_on = 1:

I termostati visualizzano l'icona della termostufa e segnalano lo stato on tramite aggiornando la tabella termostat_warming_request
eventuali condizioni di richiesta riscaldamento per impostazione oraria o per extra heating sono bypassate
la pompa di ricircolo locale è gestita secondo i timeout previsti

✅ Se stove_on = 0:

comportamento normale delle zone
stati termostati per impostazione oraria o per extra heating disponibili
caldaia attivabile come da logica standard


# 8. Sicurezze integrate
✅ Hysteresis anti‑rimbalzo
Evita cicli ON/OFF rapidi in momenti di variazione termica borderline.
✅ Timeout sensore
Se il sensore non risponde:

mantiene ultimo stato noto per sicurezza
logga l’errore
eventuale fallback dopo N errori (stufa OFF)

✅ Perdita WiFi

mantiene lo stato locale
tenta riconnessione ogni 5s
reboot dell'MCU al protrarsi dell'anomalia e spegnimento di ogni stato nel DB e della circuiteria locale (relè pompo di ricircolo)
lo stato sarà riallineato al DB e alla condizione della termostufa al ripristino


#9. Logging e analisi su heating_state e seriale
La tabella heating_state integra:

timestamp precisi degli eventi
stato on/off
motivazioni (reason)
stato della caldaia e pompa al momento del log
eventuali override (extra heating, test manuali…)

Questi log permettono di:
✅ ricostruire il comportamento reale del sistema
✅ comprendere l’apporto termico della stufa
✅ migliorare algoritmi futuri (es. adattivi)

# 10. Estensioni future

sensore duale ingresso/uscita scambiatore
curva di efficienza termica nel tempo
integrazione con misuratore potenza pompa
rilevazione “fumo” stufa per stato ON più rapido
pagina web locale di monitoraggio
