Configurazione tecnica degli apparati del sistema Home Heating IoT
Questo documento descrive nel dettaglio la configurazione tecnica degli apparati che costituiscono il sistema IoT per la gestione del riscaldamento domestico: termostati, attuatori, controller della caldaia, sensori esterni e il server centrale.

È un approfondimento complementare all'articolo introduttivo e fornisce una base tecnica per la parte hardware, firmware e architetturale.

1. Termostati ESP8266 ed ESP32
I termostati sono il punto di contatto principale con l’impianto.
Misurano le variabili ambientali, applicano la configurazione oraria, visualizzano grafici e inviano al sistema centrale lo stato della zona.

1.1 Versioni hardware disponibili
✅ Modello con display 1.8” (ESP8266 – 12E)
Utilizzato in camere e bagni
Display piccolo, comandi tramite tasti
Rilievo temperatura e umidità
Grafico 24 ore
Simboli di stato (fiamma rossa/gialla/stufa)
Aggiornamento DB ogni 10 minuti
✅ Modello con display 3.5” touch (ESP32)
Utilizzato in soggiorni e terrazzo
Display ampio con interfaccia touch
Sensori aggiuntivi: CO₂, TVOC
Previsioni meteo 6 ore
Visualizzazione stato di tutte le zone
UI più ricca e fluida grazie a FreeRTOS
1.2 Sensori collegati
Tipologie usate nel progetto:

DHT22 → temperatura/umidità (modelli sospesi ESP8266)
BME280 → temperatura/umidità/pressione
Sensore MQ-135 → qualità dell’aria (CO₂/TVOC)
Collegamento via GPIO con pull‑up dove richiesto
1.3 Logica di controllo: WARM_STATE e WARM_FORCE
Ogni termostato utilizza due flag di controllo:

WARM_STATE
Determina se la stanza deve essere scaldata in base alla configurazione oraria.

WARM_FORCE
Imposta manualmente:

0 = Normal (segue WARM_STATE)
1 = Forced ON
2 = Forced OFF
Combinazione dei due valori → determina la richiesta finale al DB.

1.4 Interfaccia utente (icone e logiche)
Ogni termostato visualizza un'icona specifica per la fonte del calore:

🔥 Fiamma rossa → Richiesta locale in base alla configurazione oraria
🟡 Fiamma gialla → Extra Heating (energia fotovoltaica)
🪵 Icona stufa → Riscaldamento da termostufa
Le modalità Extra Heating e Termostufa forzano il riscaldamento in tutte le zone,
ma interrompono automaticamente quando la temperatura locale supera 22°C.

1.5 Parallelizzazione con FreeRTOS (ESP32)
Per garantire fluidità, affidabilità e reattività:

lettura sensori → task alta priorità
aggiornamento display → task media priorità
comunicazione con DB → task bassa priorità
logiche interne → task media
Questo evita blocchi e ritardi, soprattutto con display touch e grafici.

2. Controller elettrovalvole (ESP8266 Lolin D1 Mini Lite)
Questi controller gestiscono fisicamente l’apertura/chiusura delle valvole per ogni circuito del riscaldamento a pavimento.

2.1 Funzioni principali
Leggono dallo stato centrale (tabella zone_status)
Attivano/disattivano elettrovalvole tramite relè
Applicano un timeout di sicurezza (~90 secondi) in base alla velocità di apertuta/chiusura delle elettrovalvole
Segnalano al DB lo stato aggiornato
Gestiscono 2‒4 zone a seconda della centralina
3. Controller Caldaia / Pompa di calore
Controlla:

accensione caldaia
attivazione pompa di ricircolo
interazione con termostufa
sequenze temporali per evitare shock termici
Logica generale:
Se almeno una zona richiede calore → attiva pompa di ricircolo
Dopo 30 secondi → accende la caldaia
Se la termostufa è attiva → accende comunque la caldaia che si accorge della presenza della termostufa per dal calore dell'acqua del serbatotio
In spegnimento → ferma prima la caldaia, poi la pompa
4. Controller Termostufa
La termostufa fornisce calore al circuito tramite:

scambiatore di calore condiviso
pompa di ricircolo dedicata
sensore temperatura acqua
Quando l’acqua supera la soglia (60°C):

chiude un contatto
il controller attiva la pompa di ricircolo
forza le zone in riscaldamento (icona stufa)
rispetta sempre il limite dei 22°C nelle stanze
5. Sensore esterno (ESP8266 Wemos D1 Mini Pro)
Rileva ogni 5 secondi:

temperatura
umidità
pressione (se disponibile)
E invia al DB la media ogni 10 minuti.

È usato per:

modelli adattativi
ottimizzazione Extra Heating
visualizzazione sui termostati
6. Comunicazione tra dispositivi
Tutto il sistema comunica tramite:

WiFi locale
richieste HTTP / query SQL dirette
tabelle ottimizzate per il polling (*_now)
trigger che aggiornano lo stato appena un termostato invia dati
Non viene utilizzata alcuna piattaforma cloud.

7. Raspberry Pi 2B – Server centrale
Funzioni:
MariaDB
Web server locale (dashboard)
Algoritmi Python
Aggregazione dati storici
Previsioni meteo
API Solaredge (per Extra Heating)
8. Algoritmi principali
🔶 Extra Heating
Attiva il riscaldamento in presenza di surplus fotovoltaico.

🔶 Logica adattativa
Basata su:

abitudini termiche della casa
tempo necessario a raggiungere la temperatura
condizioni meteo
🔶 Logica termostufa
Gestisce l’apporto della stufa senza creare shock termici.

9. Prossimi documenti tecnici
Nelle prossime sezioni verranno aggiunti:

pinout dettagliati
schemi elettrici completi
codice firmware degli ESP
struttura SQL del database
diagrammi architetturali del sistema
