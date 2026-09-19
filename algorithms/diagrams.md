# Automation Flowcharts – Home Heating IoT Decision Logic

🇮🇹 *Per la versione in italiano, clicca [qui](diagrams.it.md).*

This document compiles the core structural flowcharts governing the decision-making runtime loops of the Home Heating IoT architecture.

These maps are explicitly presented at a **high architectural level** to illustrate the logical operations, state validation transitions, and energy prioritizations without diving into low-level Python or firmware programming code syntax.

---

## 1. General System Layout

This flowchart maps the strict database-mediated segregation between sensory inputs, data persistence, centralized core analytics, and mechanical relay actuation loops.

```text
                                ┌───────────────┐
                                │   Ambient     │
                                │  Thermostats  │
                                │(ESP8266/ESP32)│
                                └───────┬───────┘
                                        │        
                                        │ SQL INSERT / UPDATE        
                                        ▼        
                           ┌──────────────────────────┐
                           │      MariaDB Layer       │
                           │   (Caching + Triggers)   │
                           └───┬──────────────────┬───┘
                               │                  │        
                               │                  │        
                               ▼                  ▼        
                        ┌─────────────┐  ┌──────────────────┐
                        │  Manifold   │  │ Central Engine   │
                        │  Actuators  │  │  (Raspberry Pi)  │
                        └──────┬──────┘  └────────┬─────────┘
                               │                  │
                               ▼                  ▼
                        ┌─────────────┐   ┌───────────────┐
                        │ Hydronic    │   │ Extra Heating │
                        │ Plant Relays│   │ Biomass Stove │
                        └─────────────┘   └───────────────┘
```

---

## 2. Extra Heating Optimization Algorithm

The virtual battery redirection sequence evaluates local variables metrics systematically to maximize solar self-consumption during the daily production window.

```text
                          ┌────────────────────────────┐
                          │ Execution Loop (Every 5m)  │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐   
                          │ Inside Solar Window?       │   
                          │ (09:30 – 16:30 Target)     │─────── NO ───────┐
                          └──────────────┬─────────────┘                  │
                                         │ YES                            ▼
                                         ▼                    ┌──────────────────────┐
                          ┌────────────────────────────┐      │  Extra Heating OFF   │
                          │ Query Live PV Generation   │      └──────────────────────┘
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Query Live Household Load  │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Compute Solar Net Surplus  │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Battery SoC Completed?     │─────── NO ───────┐
                          │ (7kWh Storage Validated)   │                  │
                          └──────────────┬─────────────┘                  │
                                         │ YES                            ▼
                                         ▼                    ┌──────────────────────┐
                          ┌────────────────────────────┐      │  Extra Heating OFF   │
                          │ Extra Heating = ON         │      └──────────────────────┘
                          │ Drive Global Zone Cache    │
                          └────────────────────────────┘
```

---

## 3. Biomass Thermo-Stove Priority Logic

When active, the high-temperature biomass loops override standard parameters and dynamically suppress heat pump activation while establishing an automated protection constraint.

```text
                          ┌────────────────────────────┐
                          │ Intercept Thermo-Stove State│
                          │ (Edge Node Loop Status)    │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ stove_on = 1 Validated?    │─────── NO ───────┐
                          │ (Jacket Curve >= 60°C)     │                  │
                          └──────────────┬─────────────┘                  │
                                         │ YES                            ▼
                                         ▼                    ┌──────────────────────┐
                          ┌────────────────────────────┐      │ Standard Scheduling  │
                          │ Set termostatSetup f1 = 1  │      │ Set termostatSetup f1│
                          └──────────────┬─────────────┘      └──────────────────────┘
                                         │                                │
                                         └────────────────┬───────────────┘
                                                          │
                                                          ▼
                                      ┌───────────────────────────────────┐
                                      │  Execute Decoupled Hydro Loops   │
                                      │  Driven by Ambient Overrides      │
                                      └───────────────────────────────────┘
```

---

## 4. Primary Plant Drive Logic (Boiler + Pumps)

The central runtime engine runs a strict structural protection cycle every 90 seconds to evaluate the aggregated load data from the structural zone indicators.

```text
                          ┌────────────────────────────┐
                          │ Loop Sequence (Every 90s)  │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Parse zone_status Data     │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Active Zones Detected >= 1?│─────── NO ───────┐
                          └──────────────┬─────────────┘                  │
                                         │ YES                            ▼
                                         ▼                    ┌──────────────────────┐
                          ┌────────────────────────────┐      │ Trip Burner/HP = OFF │
                          │ Recirculation Pump = ON    │      │ Timeout Delay Driven │
                          └──────────────┬─────────────┘      │ Manifold Pump = OFF  │
                                         │                    └──────────────────────┘
                                         ▼
                          ┌────────────────────────────┐
                          │ Exchanger Pre-Circulation  │
                          │ Protection Loop (90s Delay)│
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Plant Burner / HP = ON     │
                          └────────────────────────────┘
```

---

## 5. Thermostat-Actuator Zone Interaction Loop

Each independent ambient sensing terminal processes scheduling data, hardware flags, and user boundaries to determine physical activation calls.

```text
                          ┌────────────────────────────┐
                          │ Local Room Thermostat Node │
                          │ (Schedules & Manual Inputs)│
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Telemetry < Target Limit?  │─────── NO ───────┐
                          └──────────────┬─────────────┘                  │
                                         │ YES                            ▼
                                         ▼                    ┌──────────────────────┐
                          ┌────────────────────────────┐      │ Hydronic Zone Output │
                          │ Ambient < Scheduled Value  │      │ State = Inactive     │
                          │             OR             │      └──────────────────────┘
                          │ Flag f0 Active (Solar)     │
                          │             OR             │
                          │ Flag f1 Active (Biomass)   │
                          └──────────────┬─────────────┘
                                         │
                                         ▼
                          ┌────────────────────────────┐
                          │ Valve State Drive = ON     │
                          │ heating_status Record = 1  │
                          └────────────────────────────┘
```
6. Architecture Rationale SummaryThe logical state transitions mapped above secure several engineering features for the residential plant:Decoupled Isolation: Sensor arrays, logic engines, and relays function independently, preventing point-of-failure cascade events.Deterministic Behavior: Eliminates ambiguous operational profiles, shielding physical assets against component wear.Pure Sovereignty: Shifting logical structures onto the central MariaDB server leaves peripherals free to handle basic sensing routines.
