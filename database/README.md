
# Database Layer: Architecture & Persistence

This directory contains the database schemas, triggers, and architectural documentation for the central storage engine of the Home Heating IoT ecosystem.

## 📌 Directory Structure
* **[overview.md](overview.md):** Detailed analysis of the database engine selection (MariaDB), hardware host telemetry, and overall architecture.
* **[schema.md](schema.md):** Complete definition of the database structures (`Temperature` and `weather`), tables, and data models.
* **[triggers.md](triggers.md):** Production-ready SQL DDL for database triggers designed to eliminate edge-node polling latency.

## ⚙️ Core Architecture Concept
To prevent the distributed microcontroller nodes (ESP32 and ESP8266) from executing resource-heavy SQL calculations or causing network congestion, the storage layer shifts computational complexity to the central server using automated **SQL Triggers**. 

Instead of parsing millions of historical rows, edge nodes query dedicated, highly optimized `_now` cache tables maintained instantly by the database engine upon every data insertion event.

