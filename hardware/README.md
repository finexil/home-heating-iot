
# Hardware del sistema Home Heating IoT

Questa cartella contiene tutti gli schemi elettrici e i riferimenti hardware degli apparati:

## 📟 Termostati
- ![ESP8266 con display 1.8”](https://raw.githubusercontent.com/finexil/home-heating-iot/main/hardware/ESP8266_Controller_Elettrovalvole.png))
- ESP32 con display 3.5” touch

## 🔧 Controller
- Controller elettrovalvole (ESP8266 Lolin D1 Mini Lite)
- Controller caldaia
- Controller termostufa

## 🔋 Alimentazione
L'alimentatore di tutti gli elementi (Termostati e Controller) è centralizzata tramite un alimentatore switching che eroga 5V e 12V con corrente massima di 5A. Tutte le schede sono collegate all'alimentatore con cavo multicoppia steso in canalina. Questa soluzione permette di evitare l'uso di batterie per ogni sistema locale.

## 🖼️ Schemi elettrici
Gli schemi sono presenti nella sotto-cartella `hardware/` e possono essere inclusi nella documentazione tecnica.
