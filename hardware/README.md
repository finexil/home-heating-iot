# Infrastructure Hardware & Electrical Blueprint Specifications

🇮🇹 *Per la versione in italiano, clicca [qui](README.md).*


This directory compiles the complete electrical blueprints, schematic layouts, component architectures, and power distribution parameters constituting the physical layer of the Home Heating IoT ecosystem.

The hardware deployment isolates processing logic from heavy load switching, using dedicated Espressif microcontrollers interfaced via custom transistor-driven circuit grids.

---

## 📟 Ambient Thermostat Terminals

These peripheral sensing stations house the ambient monitoring sensors and the graphical display layers for human-machine interaction:

### 🔹 Compact Thermostat Node (ESP8266 + 1.8" SPI Display)
*   **Target Schematics:** Deployed across secondary bedrooms and high-humidity bathroom spaces.
*   ![ESP8266 con display 1.8”]([https://githubusercontent.com](https://raw.githubusercontent.com/finexil/home-heating-iot/main/docs/images/ESP8266_Display_1_8_INCH.png))

### 🔹 Premium Touch Terminal (ESP32 + 3.5" TFT Touchscreen)
*   **Target Schematics:** Deployed across main open-plan living zones and balcony loops.
*   **Visual Blueprint Asset:**
    ![ESP32 con display 3.5” touch]([https://githubusercontent.com](https://raw.githubusercontent.com/finexil/home-heating-iot/main/docs/images/ESP32_Display_3_5_INCH_TOUCH.png))

---

## 🔧 Automation & Plant Relay Controllers

Edge actuation stations responsible for physical load execution, mechanical protective dampening, and central exchanger interlocks:

### 🔹 Hydronic Manifold Actuator Node (ESP8266 Lolin D1 Mini Lite)
*   **Target Schematics:** Drives high-voltage 230V slow-actuation zone electrovalves via isolated low-voltage transistor grids.
*   **Visual Blueprint Asset:**
    ![Controller elettrovalvole ([ESP8266 Lolin D1 Mini Lite)](https://githubusercontent.com](https://raw.githubusercontent.com/finexil/home-heating-iot/main/docs/images/ESP8266_Controller_Elettrovalvole.png))

### 🔹 Central Boiler Plant Node (ESP8266 NodeMCU 12-E)
*   **Target Schematics:** Coordinates the primary electric heat pump compressor banks and manifold distribution pump relays.
*   **Visual Blueprint Asset:**
    ![Controller caldaia]([https://githubusercontent.com](https://raw.githubusercontent.com/finexil/home-heating-iot/main/docs/images/ESP8266_Controller_caldaia.png))

### 🔹 Hybrid Biomass Thermo-Stove Monitor (ESP32 D1 Mini)
*   **Target Schematics:** Tracks high-temperature water jacket immersion variables and triggers local return pumps.
*   **Visual Blueprint Asset:**
    ![Controller termostufa]([https://githubusercontent.com](https://raw.githubusercontent.com/finexil/home-heating-iot/main/docs/images/Controller_Termostufa.jpg))

---

## 🔋 Centralized Power Distribution Network (Bus Architecture)

To secure absolute runtime availability, maximize operational lifespan, and completely remove the maintenance overhead of individual battery replacement, this architecture implements a **100% hardwired, centralized power bus**:

*   **Core Exchanger Unit:** A centralized industrial-grade switching mode power supply (SMPS).
*   **Dual-Rail Voltage Rails:** The distribution network delivers two independent electrical rails:
*   **`5 Vdc Rail (5A Peak Limit):`** Supplies logic lines, microcontrollers, environmental sensors, and TFT display backlight grids.
*   **`12 Vdc Rail (5A Peak Limit):`** Drives inductive mechanical relay coils, electrovalve pilot logic, and active ventilation (VMC) fan units.
*   **Topological Cabling Matrix:** Power and edge communications are distributed to each remote thermostat and manifold housing group through dedicated **multi-pair shielded telecom cabling** routed inside structural electrical conduits. 
*   **Local Filtering:** Peripheral node footprints integrate onboard buck converters, electrolytic decoupling capacitors, and low-dropout (LDO) linear regulators to suppress transient voltage ripples and cross-node electromagnetic noise during heavy relay switching cycles.
