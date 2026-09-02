# 🖨️ Printing Assistant (PA)

> **The cozy, local-first control plane for your entire 3D printing setup.**
> *Vendor-agnostic, privacy-focused, and seamlessly integrated into your home.*

Welcome to **Printing Assistant**! Inspired by the local-first, home-centric philosophy of Home Assistant, PA brings all your 3D printers, material spools, AI safety monitors, and enclosure sensors into one unified, inviting dashboard.

Whether you're running a single Voron, a fleet of Bambu or Prusa machines, or a custom bed-slinger in your garage, Printing Assistant gives you complete ownership over your hardware—100% cloud-free.

---

## 🌟 Why Printing Assistant?

* **🏡 100% Local & Sovereign** — Zero mandatory cloud accounts or external dependencies. Everything runs on your local network over WebSockets, MQTT, and native HTTP APIs.
* **🔌 Vendor-Agnostic Engine** — Connect Klipper, Bambu Lab, OctoPrint/Marlin, PrusaBuddy, and RepRapFirmware under one single, unified interface.
* **⚡ Event-Driven Core** — Sub-second telemetry streams, real-time G-code rendering, and instant state updates across all your devices.
* **👁️ Local Edge Vision AI** — Spaghetti detection and layer shift monitoring run locally on low-cost NPUs/GPUs (Raspberry Pi 5 NPU, Coral TPU, CUDA) with total privacy.
* **🛡️ Environmental Safety First** — Native integration with smart plugs, relays, and air filters to handle VOC scrubbing and hard power cutoffs automatically.

---

## 🏗️ Architecture Overview

Printing Assistant sits at the heart of your workshop, orchestrating hardware, raw materials, slicer pipelines, and environmental controls through a decoupled driver system.

```text
+---------------------------------------------------------------------------------------------------+
|                                  PRINTING ASSISTANT CORE ENGINE                                   |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|   [ FRONTEND LAYERS ]                                                                             |
|   ├── Web App (Vue 3 + Tailwind CSS)                                                             |
|   ├── Native Touch Kiosk (Optimized for Raspberry Pi & Wall Displays)                             |
|   └── Companion App / Mobile PWA                                                                  |
|                                                                                                   |
|   [ API & WEBSOCKET GATEWAY ] (FastAPI / Rust Async Engine - Port 8124)                           |
|   ├── REST API (G-code ingestion, fleet metadata, user management)                                |
|   ├── WebSocket Broker (Live telemetry, camera feeds, 3D G-code visualizer)                        |
|   └── Home Assistant Gateway (Native MQTT Discovery & WebSocket API)                              |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
|                                           EVENT BUS                                               |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|   [ CORE MANAGERS & ENGINES ]                                                                     |
|   ├── State Engine                   ├── Material Engine (Spoolman / NFC / Scale Sync)             |
|   ├── Dispatch Engine                ├── Edge Vision AI (Local YOLOv11 / NPU Spaghetti Monitor)   |
|   └── Automation Engine              └── Safety Orchestrator (Thermal runaway, VOC, Relays)       |
|                                                                                                   |
|   [ UNIFIED PRINTER DRIVERS ]                                                                     |
|   ├── Klipper Driver (Moonraker WS)  ├── Bambu Driver (Local MQTT & FTPS)                          |
|   ├── OctoPrint (Marlin REST/WS)     ├── Prusa Driver (PrusaBuddy API)                            |
|   └── RepRap / Duet Driver           └── Direct Serial / USB MCU Interface                        |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
|                                 +----------------------+----------------------+                   |
|                                 |   HARDWARE FLEET     | ENVIRONMENT & SENSORS|                   |
|                                 | (Klipper, Bambu, etc)| (HEPA, Smoke, Plugs) |                   |
|                                 +----------------------+----------------------+                   |

```

---

## 🧩 Core Subsystems

### A. Unified Abstraction Driver Layer

Printing Assistant translates every printer protocol into a single standardized schema (`PrinterEntity`), freeing you from vendor lock-in:

* **Klipper / Moonraker:** Native JSON-RPC over WebSockets with direct HTTP G-code upload handling.
* **Bambu Lab:** Local MQTT bridge over port 8883 (SSL) paired with direct local FTPS for `.gcode.3mf` file transfers—no Bambu Cloud needed.
* **PrusaBuddy & OctoPrint:** Direct polling and WebSocket handlers mapping telemetry seamlessly.

```json
// Internal Standardized Telemetry Schema (PrinterEntity)
{
  "entity_id": "printer.voron_2_4",
  "name": "Voron 2.4 350mm",
  "state": "printing",
  "temperatures": {
    "extruder": {"current": 245.2, "target": 245.0},
    "bed": {"current": 109.8, "target": 110.0},
    "chamber": {"current": 52.4, "target": 55.0}
  },
  "progress": {
    "percentage": 64.2,
    "current_layer": 182,
    "total_layers": 283,
    "time_remaining_sec": 4120
  },
  "active_material": {
    "slot_id": 0,
    "spool_id": "spool-abs-black-0921",
    "material": "ABS",
    "vendor": "Polymaker",
    "remaining_grams": 410.5
  }
}

```

### B. Material & Inventory Engine (Spool Engine)

Keep precise track of your filament library without extra fuss:

* **Spoolman Integration:** Bidirectional syncing with your local Spoolman instance out of the box.
* **Automatic Weight & NFC Sync:** Hooks into HX711 load-cell scales or RFID/NFC spool tags to deduct filament usage down to the gram.
* **Smart Runout Safeguards:** Cross-references print estimates against available spool weight before starting a job, offering auto-failover or swapping prompts.

### C. Local Edge Computer Vision AI

Keep an eye on your prints without streaming sensitive video to third-party clouds:

* **Local Hardware Acceleration:** Runs lightweight YOLO models on Raspberry Pi 5 NPU HATs, Coral TPUs, RK3588 boards, or local Nvidia GPUs.
* **Smart Anomaly Detection:** Evaluates camera frames every 3 seconds for spaghettiing, poor bed adhesion, and layer shifts.
* **Instant Local Action:** Fires an `event.vision.anomaly_detected` trigger to pause prints immediately and send alerts via NTFY, Gotify, or Home Assistant.

### D. Fleet Dispatch & Print Queue

Turn your scatter of printers into an organized, stress-free pool:

* **Material-Aware Routing:** Drop a file in, and PA routes it to the right machine based on nozzle diameter, bed size, chamber heating, and loaded filament.
* **Batch Production:** Easily fan out mass-production jobs across available printers in your workshop.

### E. Environmental & Safety Automation

Built to talk directly with Home Assistant, Zigbee, Tasmota, and ESPHome:

* **Air Quality Scrubbing:** Automatically fires up enclosure exhaust fans and HEPA/carbon filters when ABS, ASA, or PC prints start.
* **Hard Emergency Cutoff:** If thermal runaway or unsafe chamber temperatures are detected, PA turns off smart plugs directly to cut AC power to the machine safely.

---

## 🛠️ Technology Stack

| Layer | Technology | Why We Chose It |
| --- | --- | --- |
| **OS Stack** | **Printing Assistant OS** | Immutable Buildroot / Debian Linux image lightweight enough for Raspberry Pi & Mini PCs. |
| **Core Engine** | **Python (FastAPI) / Rust (Tokio)** | High concurrency for multi-printer WebSockets, fast event-bus response, and low footprint. |
| **Database** | **SQLite (WAL mode)** | Zero-config, rock-solid local storage for state logs, spool data, and print histories. |
| **Frontend** | **Vue 3 / Tailwind CSS** | Lightning-fast rendering with Three.js G-code previews and touch-friendly UI. |
| **Messaging** | **Embedded Mosquitto MQTT** | Effortless auto-discovery with Home Assistant and local smart devices. |

---

## 📁 Modular Directory Structure

```text
/etc/printing-assistant/
├── config.yaml          # Master System & Network Configuration
├── storage/
│   ├── pa_database.db   # SQLite Local State Engine
│   └── models/          # Edge AI Weights (.onnx / .rknn)
├── drivers/             # Pluggable Printer Adapters
│   ├── klipper.py
│   ├── bambu.py
│   └── prusa.py
└── blueprints/          # Community Safety & Automation Blueprints
    ├── abs_ventilation.yaml
    └── fire_safety_cutoff.yaml

```

---

## 🗺️ Roadmap & Next Steps

* **Phase 1: Core Engine & Adapters (MVP)**
Building the core Python/Rust event bus alongside initial Moonraker (Klipper) and Bambu MQTT driver adapters.
* **Phase 2: Spool Engine & Fleet Dispatch**
Integrating Spoolman API sync, load-cell support, and multi-printer queue routing.
* **Phase 3: Vision AI & Home Assistant Ecosystem**
Packaging ONNX local vision inferencing for SBC NPUs and publishing native MQTT discovery blueprints for Home Assistant.

---

**Built with ❤️ for the open-source 3D printing community.**

*Crafted by Pooya Mohammadhossein & contributors.*
