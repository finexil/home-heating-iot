# Schema del Database – Home Heating IoT

Il database MariaDB è il centro logico dell’intero sistema.  
Tutti i dispositivi ESP leggono e scrivono qui le informazioni necessarie per
coordinare termostati, attuatori, caldaia, termostufa e algoritmi centrali.

Il DB è organizzato in due insiemi principali:

- **Database `temperature`**
- **Database `weather`**

Questo documento descrive tutte le tabelle principali del database `temperature`.

---

# 1. Tabelle principali del database

## 1.1 `Warming_state`
Registra gli eventi di accensione/spegnimento inviati dai termostati.

| Campo | Tipo | Descrizione |
|-------|------|-------------|
| id | INT | Identificatore evento |
| room | VARCHAR | Zona (es: "pt soggiorno") |
| FLG_ON | TINYINT | 0=OFF, 1=ON |
| timestamp | DATETIME | Momento dell’evento |

Questa tabella è collegata al trigger che aggiorna `termostat_warming_request`.

---

## 1.2 `termostat_warming_request`
Contiene l’ultimo stato richiesto da ogni zona, pronto per essere letto dai controller.

| Campo | Tipo | Descrizione |
|-------|------|-------------|
| location | VARCHAR | Sempre “ceresole” nel tuo impianto |
| pt_soggiorno | TINYINT | 0/1 stato zona |
| pt_camera | TINYINT | 0/1 |
| pt_bagno  | TINYINT | 0/1 |
| p1_soggiorno | TINYINT | 0/1 |
| p1_camera | TINYINT | 0/1 |
| p1_bagno  | TINYINT | 0/1 |
| te_terrazzo | TINYINT | 0/1 |

Tabella aggiornata dai trigger al cambio di stato delle zone.

---

## 1.3 `termostat_temp_now`
Memorizza le temperature aggiornate in tempo reale per consulta veloce da parte del web server.

| Campo | Tipo |
|--------|------|
| location | VARCHAR |
| pt_soggiorno | FLOAT |
| pt_camera | FLOAT |
| pt_bagno | FLOAT |
| p1_soggiorno | FLOAT |
| p1_camera | FLOAT |
| p1_bagno | FLOAT |
| te_terrazzo | FLOAT |

Aggiornata automaticamente dai trigger collegati alla tabella `powerDetails`.

---

## 1.4 `termostat_temp_hum`
Storico delle temperature/umidità (ogni 10 minuti).

| Campo | Tipo |
|--------|------|
| id | INT |
| room | VARCHAR |
| temperature | FLOAT |
| humidity | FLOAT |
| timestamp | DATETIME |

Questa tabella contiene anni di dati utili per analisi.

---

## 1.5 `zone_status`
Usata dagli attuatori per comunicare lo stato effettivo delle elettrovalvole.

| Campo | Tipo |
|--------|------|
| id | INT |
| room | VARCHAR |
| zone_on | TINYINT |
| timestamp | DATETIME |

---

## 1.6 `heating_state`
Stato della caldaia/pompa/extra heating.

| Campo | Tipo |
|--------|------|
| id | INT |
| pump_on | TINYINT |
| boiler_on | TINYINT |
| stove_on | TINYINT |
| timestamp | DATETIME |

---

## 1.7 `external_temp_hum` e `external_temp_hum_now`
Sensori meteo esterni (ogni 5 sec / 10 min).

`external_temp_hum_now` è tenuta aggiornata da un trigger.

---

# 2. Relazione tra tabelle

- I termostati scrivono in `Warming_state` → Trigger aggiorna `termostat_warming_request`
- I termostati scrivono temperatura in `termostat_temp_hum` → Trigger aggiorna `termostat_temp_now`
- Gli attuatori leggono `termostat_warming_request`
- Gli attuatori aggiornano `zone_status`
- La caldaia legge `zone_status` e `heating_state`
- Gli algoritmi (Extra Heating, adattativo) modificano `heating_state`

---

# 3. Schema logico del flusso dati (alta sintesi)
