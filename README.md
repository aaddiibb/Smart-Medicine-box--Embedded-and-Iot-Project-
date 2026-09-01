# 💊 Smart Medicine Box

An IoT-based Smart Medicine Box powered by an **ESP32 microcontroller** and a modern **Laravel web application**. This project ensures patients never miss a dose by combining physical hardware alerts (LEDs, buzzer, sensors) with a real-time, easy-to-use web interface for scheduling and monitoring.

![Platform](https://img.shields.io/badge/platform-ESP32-blue)
![Backend](https://img.shields.io/badge/backend-Laravel-red)

---

## 📷 Preview

| | |
|:---:|:---:|
| **Dose Mode**<br>![Dose Mode](images/dosemode.jpeg) | **Medicine Mode**<br>![Medicine Mode](images/medicinemode.jpeg) |
| **Missed Dose**<br>![Hardware](images/missed.jpeg) | **Hardware**<br>![Hardware](images/box.jpeg) |
---

##  Features

- **Two Operating Modes**
  - **Dose Mode** — Schedule reminders per physical compartment (e.g., Morning Dose, Evening Dose), one time slot per compartment.
  - **Medicine Mode** — Group any subset of the 6 compartments under a named medicine and schedule it independently, with support for up to 20 named medicines.
- **Physical Alerts** — An active buzzer and per-compartment LEDs guide the user directly to the medicine they need to take.
- **Smart Sensors**
  - A **reed switch** detects when the box lid is opened.
  - **6× IR obstacle sensors** (one per compartment) detect when medicine has been physically removed, using a baseline-comparison technique to avoid false positives from empty compartments.
- **Two-Phase Detection Logic** — Waits for the lid to open (Phase 1), then monitors each pending compartment's IR sensor against a captured baseline until all medicine is collected (Phase 2).
- **Missed Dose Tracking** — Automatically detects and logs doses not collected within a configurable timeout window, recording the exact compartments that were missed.
- **Resilient Log Delivery** — Since the ESP32 never initiates outbound requests, missed-dose events are buffered on-device and piggybacked onto the next status poll, then acknowledged by the web app — so no data is lost even if the web app is briefly down.
- **Real-Time Web Dashboard** — A Tailwind CSS–powered Laravel app for configuring schedules, viewing live box status (polled every 5s), and reviewing/clearing missed-dose history.
- **Device Utilities** — Sync the ESP32's clock directly from the browser, remotely restart the device, and test connectivity from the dashboard.

---

##  System Architecture

```
┌─────────────────────┐        HTTP / JSON          ┌──────────────────────┐
│   Laravel Web App   │  ────────────────────────▶ │   ESP32 WebServer     │
│  (Tailwind CSS UI)  │  ◀──────────────────────── │   (medicine_box.ino)  │
└─────────────────────┘   polled every 5 seconds    └──────────┬───────────┘
                                                               │
                                                   Digital I/O │
                                                               ▼
                                              ┌─────────────────────────────┐
                                              │  Hardware Layer             │
                                              │  • 6× LEDs (per compartment)│
                                              │  • 6× IR sensors            │
                                              │  • 1× Reed switch (lid)     │
                                              │  • 1× Active buzzer         │
                                              └─────────────────────────────┘
```

**Data flow:** Web App (Laravel) → HTTP/JSON → ESP32 WebServer → Hardware (LEDs, Buzzer, Reed Switch, IR Sensors) → Status → HTTP/JSON → Web App (polled every 5s)

The ESP32 always acts as the **HTTP server**; it never opens outbound connections. This is why missed-dose logs use a **poll-and-piggyback** pattern instead of a push/webhook — the device queues log entries in RAM and the web app collects them on its next `/status` poll, then explicitly acknowledges receipt via `/clear-logs`.

---

##  Hardware Requirements

| Component | Quantity | Notes |
|---|---|---|
| ESP32 Development Board | 1 | Hosts the WebServer and all control logic |
| IR Obstacle Sensor (FC-51 or similar) | 6 | One per compartment, detects medicine removal |
| 5mm LED | 6 | One per compartment, lights up during a reminder |
| Magnetic Reed Switch (Normally Open) | 1 | Detects when the box lid is opened |
| Active Buzzer (3.3V) | 1 | Audible alert; driven with `digitalWrite` for full volume |
| Jumper wires, resistors, enclosure | — | Standard prototyping hardware |

### Pin Configuration

| Constant | Values | Purpose |
|---|---|---|
| `IR_PINS[6]` | `{15, 19, 4, 16, 17, 5}` | IR sensor OUT pins, compartments 1→6 |
| `LED_PINS[6]` | `{18, 2, 21, 25, 33, 32}` | LED pins, compartments 1→6 |
| `REED_PIN` | `22` | Lid reed switch (`INPUT_PULLUP`) |
| `BUZZER_PIN` | `23` | Active buzzer output |
| `REED_TAKEN_STATE` | `HIGH` | Pin state meaning "lid open" |

All hardware pins are defined as named constants at the top of the firmware, so rewiring only ever requires editing this one section.

---

##  Software Stack

- **Firmware:** C++ (Arduino framework) on ESP32, using `WebServer` and `ArduinoJson`
- **Backend:** Laravel (PHP), SQLite database
- **Frontend:** Blade templates + Tailwind CSS
- **Communication:** REST-style HTTP/JSON between Laravel and the ESP32, polled every 5 seconds

### Key Firmware Design Points

- **Two-phase pickup detection:** Phase 1 waits for the reed switch to signal the lid opening, then captures a baseline IR reading for every compartment. Phase 2 compares each compartment's live IR reading against that baseline — a *change* means the medicine was removed. This baseline approach avoids false "taken" readings from compartments that read HIGH even when empty.
- **Non-blocking time reads:** `getHHMM()` calls `getLocalTime()` with a 0ms timeout so the HTTP server never freezes waiting on an unsynced clock.
- **Multi-compartment firing:** `checkSchedules()` collects *all* matching compartments for the current minute before triggering a single combined reminder, and de-duplicates compartments shared across multiple medicines in Medicine Mode.
- **Missed-dose logging:** On timeout, an event is encoded as `"mode|HH:MM|comp1,comp2"` and buffered in a 10-slot RAM ring buffer (`g_logQueue`) until the web app retrieves and acknowledges it.

---

##  HTTP API Reference

| Endpoint | Method | Called by | Effect on device |
|---|---|---|---|
| `/ping` | GET | Connection test | Returns `{pong:true}` |
| `/status` | GET | Live Status page (every 5s) | Returns mode, status, timeout, and any pending `log_queue` entries |
| `/set-mode` | POST | Mode Selection page | Switches operating mode, resets status to "Ready" |
| `/set-dose-schedules` | POST | Dose Mode save | Overwrites the 6 dose-mode compartment slots |
| `/set-medicine-schedules` | POST | Medicine Mode "Save All" | Overwrites all named medicine schedules |
| `/set-timeout` | POST | Settings page | Updates the missed-dose timeout window |
| `/sync-time` | POST | Device Controls | Sets the ESP32's system clock from the browser's time |
| `/restart` | POST | Device Controls | Acknowledges, then reboots the ESP32 |
| `/clear-logs` | POST | After log persistence | Clears the device's in-RAM missed-dose buffer |

---

## 🗂️ Application Structure (Laravel)

| Controller | Responsibility |
|---|---|
| `DoseModeController` | Validates and saves the 6-slot dose schedule to SQLite, then pushes a trimmed payload to the ESP32 |
| `MedicineModeController` | Full CRUD for named medicines; `sync()` pushes all medicines to the ESP32 in one call |
| `MissedDoseLogController` | Paginated view of missed-dose history (`index()`) and a destructive `clear()` action |
| `LiveStatusController` | Polls `/status` every 5s, persists any `log_queue` entries, and acknowledges via `/clear-logs` |

### Missed-Dose Logging Pipeline

1. Device detects a timeout → writes an entry to its in-RAM `log_queue`.
2. Web app polls `GET /status` → reads `log_queue` → inserts rows into the `missed_dose_logs` table.
3. Web app calls `POST /clear-logs` → device resets its buffer.
4. The Missed Doses page reads persisted logs via `MissedDoseLogController::index()`.

---

##  Getting Started

### 1. Flash the Firmware
1. Open `esp_32.ino` in the Arduino IDE.
2. Install the required libraries: `ArduinoJson`, `WiFi`, `WebServer`.
3. Configure your WiFi credentials, static IP, and timezone offset at the top of the file.
4. Wire up the hardware per the [pin configuration](#pin-configuration) table above.
5. Flash to your ESP32 and open the Serial Monitor to confirm it connects and prints its IP address.

### 2. Set Up the Laravel App
```bash
git clone <your-repo-url>
cd smart-medicine-box

# Install dependencies
composer install
npm install

# Environment setup
cp .env.example .env
php artisan key:generate

# Database
php artisan migrate

# Build frontend assets
npm run build
```

### 3. Run the App
```bash
php artisan serve
```

### 4. Connect and Configure
1. Open the web app in your browser.
2. Enter your ESP32's IP address in the Device Controls page and confirm the connection (`/ping`).
3. Sync the device clock (`/sync-time`).
4. Choose a mode (Dose Mode or Medicine Mode) and set up your first schedule.
5. Monitor live status and missed doses from the dashboard.

---


