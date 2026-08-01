# Printing-Assistant
3D Printing open source project

Here is a architectural draft for Printing Assistant—a local-first, vendor-agnostic, open-source 3D printing control plane built on the design patterns of Home Assistant.
Executive Summary & Core Guiding Principles
Printing Assistant (PA) acts as a central orchestrator for additive manufacturing hardware, raw materials, slicer automation, and environmental safety.
 * 100% Local-First & Sovereign: Zero mandatory cloud dependencies. Operations execute on local subnets via WebSocket, MQTT, and native HTTP APIs.
 * Decoupled Control & Translation Layers: A unified state engine interfaces with any printer (Klipper, Bambu, OctoPrint/Marlin, PrusaBuddy, RepRapFirmware) via adapter drivers.
 * Event-Driven Architecture: Core operations (state changes, camera inferencing, sensor triggers) stream over a central Event Bus.
 * Privacy-Preserving Edge Compute: Computer vision (layer shifts, spaghetti detection) runs on local NPUs/GPUs using lightweight YOLO/OpenCV pipelines.
1. High-Level System Architecture
+---------------------------------------------------------------------------------------------------+
|                                  PRINTING ASSISTANT CORE ENGINE                                   |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|  [ FRONTEND LAYERS ]                                                                             |
|    ├── Web App (Vue/React + Tailwind)                                                             |
|    ├── Native Kiosk Mode (Touch UI for Wall-Mounted / Raspberry Pi Displays)                       |
|    └── Mobile Client (Progressive Web App / Flutter)                                              |
|                                                                                                   |
|  [ API & WEBSOCKET GATEWAY ] (FastAPI / Rust Async Engine - Port 8124)                             |
|    ├── REST API (G-code ingestion, fleet metadata, user management)                               |
|    ├── WebSocket Broker (Sub-second telemetry streams, camera feeds, live G-code renderers)        |
|    └── Home Assistant Integration (WebSocket API / Native MQTT Discovery)                          |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
|                                           EVENT BUS                                               |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|  [ CORE MANAGERS & ENGINES ]                                                                      |
|    ├── State Engine          ├── Material Engine (Spoolman / NFC / Weight Scale Sync)             |
|    ├── Dispatch Engine       ├── Edge Vision AI (Local YOLOv11 / NPU Spaghetti & Layer Shift)     |
|    └── Automation Engine     └── Safety Orchestrator (Thermal runaway, VOC/Air, Relays)          |
|                                                                                                   |
|  [ ABSTRACTION & DRIVER LAYER (Unified Printer API) ]                                              |
|    ├── Klipper Driver (Moonraker WS)         ├── Bambu Driver (Local MQTT & FTPS)                 |
|    ├── OctoPrint Driver (Marlin REST/WS)     ├── Prusa Driver (PrusaBuddy API)                    |
|    └── Custom RepRap / Duet Driver           └── Serial / Direct USB MCU Interface                |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
                                              |
                       +----------------------+----------------------+
                       |                                             |
            [ HARDWARE FLEET ]                             [ ENVIRONMENT & SENSORS ]
         (Klipper, Bambu, Prusa, etc.)                  (HEPA Filters, Smoke Detectors,
                                                         Smart Plugs, VOC Sensors)

2. Core Subsystems Breakdown
A. Unified Abstraction Driver Layer
To eliminate vendor lock-in, Printing Assistant translates external protocol dialects into a single, standardized internal data schema: PrinterEntity.
 * Klipper / Moonraker Adapter: Native JSON-RPC over WebSocket for telemetry; HTTP endpoint streaming for G-code uploads.
 * Bambu Lab Adapter: Internal MQTT bridge listening on port 8883 (SSL) paired with direct local FTPS for sending .gcode.3mf directly to machines on the local LAN without Bambu Cloud.
 * PrusaBuddy / OctoPrint Adapters: REST/WS polling handlers mapping telemetry back to the central engine.
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

B. Material & Inventory Engine (Spool Engine)
Replaces manual tracking and isolated databases with a local material catalog:
 * Native Spoolman API Sync: Deep bidirectional integration with open-source Spoolman instances.
 * Automated Weight / NFC Deduct: Listens to load-cell hardware scales (HX711) or RFID/NFC tag readers mounted on spool holders to update remaining filament weight down to the gram in real time.
 * Smart Runout Safeguard: Before sending a print job, the Dispatch Engine cross-references the gcode material estimate against available spool weight. If insufficient, it either triggers a multi-spool failover or prompts for a spool swap.
C. Local Edge Computer Vision AI
Cloud-based AI monitoring (such as Obico Cloud) is replaced with a local-first, low-latency vision pipeline:
 * Hardware Acceleration: Runs light-weight YOLO models optimized for local NPUs (Raspberry Pi 5 NPU HAT, Coral TPU, RK3588, or local Nvidia CUDA).
 * Inference Loop: Takes frame grabs via RTSP/MJPEG streams every 3 seconds during active prints.
 * Anomaly Detection: Evaluates frames for:
   * Spaghetti / Loss of Adhesion
   * Layer Shifting (cross-referencing current Z-height against camera keyframes)
   * First Layer Defects
 * Action Logic: Emits an event.vision.anomaly_detected to the Event Bus, allowing the system to immediately pause the print locally and notify the user via local push services (Gotify, NTFY, or Home Assistant).
D. Fleet Dispatch & Print Queue Orchestrator
Converts an isolated machine array into an automated manufacturing pool:
 * Material-Aware Scheduling: User drops a G-code/3MF file into Printing Assistant. The engine scans machine states and routes the file to the optimal printer based on nozzle size, chamber enclosure capabilities, build plate dimensions, and loaded filament type.
 * Print Farm Batch Management: Automatically distributes mass-production requests across multiple available printers simultaneously.
E. Environmental & Safety Automation Engine
Interactions with smart home environments work out of the box through native Home Assistant integration (via MQTT Discovery or REST/WS API):
 * Toxic Filament Air Scrubbing: When an ABS/ASA/PC print initializes, PA triggers smart relays to power on enclosure exhaust fans and active carbon/HEPA filtration systems.
 * Thermal Safety Shutdown: If thermal runaway is detected or chamber temperatures exceed safety limits, PA triggers local smart plugs (Zigbee/Tasmota/ESPHome) to cut hard A/C mains power to the machine, bypassing unresponsive MCU boards.
3. Deployment & Technology Stack
| Component Layer | Technology Recommendation | Rationale |
|---|---|---|
| OS Stack | Printing Assistant OS (PAOS) | Immutable Buildroot / Debian-based Linux image tailored for Single Board Computers (Raspberry Pi 4/5, x86 Mini PCs, Orange Pi). |
| Core Engine / Async Broker | Python (FastAPI) or Rust (Tokio) | High concurrency for multi-printer WebSockets, event-bus performance, and fast I/O processing. |
| Database & Caching | SQLite (WAL mode) + Embedded Key-Value | Zero-configuration local database for state logging, spool data, and print job histories. |
| Frontend Framework | Vue.js 3 / Web Components | Fast rendering for G-code 3D preview canvases (Three.js/Open3D), mobile responsive, low footprint. |
| Local Messaging | Embedded Mosquitto MQTT & Internal Event Bus | Seamless discovery with Home Assistant, ESPHome sensors, and third-party scripts. |
4. Proposed Modular Data Directory Structure
/etc/printing-assistant/
├── config.yaml               # Master System & Network Config
├── storage/
│   ├── pa_database.db        # SQLite Local State DB
│   └── models/               # Local Edge AI Weights (.onnx / .rknn)
├── drivers/                  # Pluggable Printer Adapters
│   ├── klipper.py
│   ├── bambu.py
│   └── prusa.py
└── blueprints/               # Community Automation Blueprints
    ├── abs_ventilation.yaml
    └── fire_safety_cutoff.yaml

Next Steps for Implementation
 * Phase 1 (MVP Core): Build the Python/Rust event bus alongside the Moonraker (Klipper) and Bambu MQTT driver adapters.
 * Phase 2 (Material & Fleet Dispatch): Implement Spoolman API syncing and multi-machine job queues.
 * Phase 3 (Vision AI & Safety Integrations): Package ONNX-based local vision inferencing for low-cost NPU hardware and expose native MQTT discovery nodes for Home Assistant.
