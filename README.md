![logo](https://github.com/user-attachments/assets/db8cac1d-6cd5-49cf-8b4a-c0d7abeebb8f)

# XeWe LED - Web — control an ESP32 LED strip from any browser

San José State University team project · Fall 2024 (2024-10-21 → 2024-12-04) · Team: Max Dokukin, Robin Goswami ([robin470](https://github.com/robin470)) · Status: Completed (proof of concept)

## Overview

This project (called **LUMN** in its UI, "LED Lights Controller via Web" in the final report) is a Flask-based web
server for controlling an addressable LED strip driven by an ESP32-C3 microcontroller. A one-page web interface sets
the strip's color, brightness, on/off state and mode (Solid / Fade); the server keeps that state in memory and exposes
it as a small JSON API, and the ESP32-C3 pulls `GET /get_data` every 2 seconds and applies the result to the LEDs. The
server also generates a QR code linking to itself for easy access from a phone. It was packaged with Docker and
deployed to Google Cloud Run (`us-west1`), so the lights can be controlled from anywhere the page loads.

A second iteration of the interface — multi-device, iro.js color sliders, per-device state API — lives in the private
repo [`xewe-labs/xewe-led-web-ui`](https://github.com/xewe-labs/xewe-led-web-ui).

## Highlights

- 4 Flask routes in 76 lines (`app.py`): the page, `POST /get_info` (write state), `GET /get_data` (read state), `GET /qr_code` (PNG QR code of the site URL)
- 6-field LED state — `r`, `g`, `b`, `brightness`, `state`, `mode` — shared by the browser and the device over plain JSON
- Browser-side JavaScript converts the color picker's hex value to RGB, posts the state with `fetch`, and restores the controls from `/get_data` on page load (`templates/index.html`)
- Containerised on `python:3.12.5` and deployed to Google Cloud Run with four commands (`deployment_cmd.txt`); a demo video shows the page served from a `run.app` URL driving the strip
- 32 commits across two contributors, two merged pull requests (2024-10-21 → 2024-12-04, plus later housekeeping)

## How it works

![Web interface](static/media/resources/ui.webp)

```
browser (color picker, switches, slider) ── POST /get_info (JSON) ──▶ Flask on Cloud Run ── in-memory state
ESP32-C3 firmware ── GET /get_data every 2 s ◀──────────────────────────────┘ ──▶ LED strip
```

- **Flask app** (`app.py`) — keeps a single in-memory `devices` dict; `/get_info` merges any subset of the six fields
  (missing fields keep their previous value, `state`/`mode` are coerced to 0 or 1); `/get_data` returns the current
  state with defaults (white, brightness 255, on, Solid).
- **Web page** (`templates/base.html`, `templates/index.html`, `static/styles.css`) — Jinja template with a color
  picker, an ON/OFF switch, a 0–255 brightness slider shown as a percentage, a Solid/Fade mode switch and a "Set LED"
  button; dark theme with cyan headings.
- **QR code** (`GET /qr_code`) — builds a version-1 QR code (error correction L, box size 10, border 4) of
  `request.url_root` with `qrcode` + Pillow and streams the PNG from memory.
- **Container** (`Dockerfile`) — copies the repo into `python:3.12.5`, installs the pinned requirements and runs
  `flask run --host=0.0.0.0 --port=5000`.
- **ESP32-C3 firmware** — polls `GET /get_data` every 2 seconds and applies the result; not included in this repo
  (custom firmware and the hardware schematic were available on request).

### API endpoints

#### 1. `POST /get_info`
- **Description**: Updates the LED configuration (any subset of fields).
- **Input**: JSON payload with the following fields:
  - `r` (int): Red intensity (0–255).
  - `g` (int): Green intensity (0–255).
  - `b` (int): Blue intensity (0–255).
  - `brightness` (int): Brightness level (the web UI sends the raw 0–255 slider value).
  - `state` (int): LED state (1 for ON, 0 for OFF).
  - `mode` (int): Operating mode (1 for Fade / "Automatic", 0 for Solid / "Manual").
- **Response**:
  ```json
  {
    "success": true,
    "message": "Data updated successfully"
  }
  ```

#### 2. `GET /get_data`
- **Description**: Retrieves the current LED settings (polled by the ESP32-C3).
- **Response**:
  ```json
  {
    "r": 255,
    "g": 255,
    "b": 255,
    "brightness": 255,
    "state": 1,
    "mode": 0
  }
  ```

#### 3. `GET /qr_code`
- **Description**: Generates and serves a PNG QR code linking to the web server.

#### 4. `GET /` (and `POST /`)
- **Description**: Renders the control page.

## Results

| Metric | Value | Note |
|---|---|---|
| HTTP routes | 4 | `app.py` (76 lines) |
| Shared LED state | 6 fields | `r`, `g`, `b`, `brightness`, `state`, `mode` |
| Device update interval | 2 s | ESP32-C3 polling `GET /get_data` (firmware not in this repo) |
| Deployment | Google Cloud Run, `us-west1` | service `esp-web-server`, image in Artifact Registry |
| Commits | 32 (Max 21, robin470 11) | `git log`, 2024-10-21 → 2026-04-22 incl. housekeeping |

The project is a working proof of concept: the web page, the cloud-hosted state and the ESP32-C3 closed the loop
(see the demo video `static/media/resources/IMG_1993_optimized.mp4` — slower than usual due to slow Wi-Fi). There is no
automated test suite and no quantitative benchmark.

## Getting started

Prerequisites: Python 3.7+ (the container uses 3.12.5), Docker, the Google Cloud SDK for deployment, and an ESP32-C3
with compatible firmware and an LED strip for the hardware side.

```bash
# local
pip install -r requirements.txt
python app.py            # open http://127.0.0.1:5000
```

### Google Cloud deployment

```bash
# 1. Set the Google Cloud project
gcloud config set project esp-led-server

# 2. Build the Docker image
docker buildx build --platform linux/amd64 -t us-west1-docker.pkg.dev/esp-led-server/esp-led-server/esp-led-server:latest .

# 3. Push the Docker image
docker push us-west1-docker.pkg.dev/esp-led-server/esp-led-server/esp-led-server:latest

# 4. Deploy with Google Cloud Run
gcloud beta run deploy esp-web-server \
    --image us-west1-docker.pkg.dev/esp-led-server/esp-led-server/esp-led-server:latest \
    --region us-west1

# Stop traffic
gcloud run services update esp-web-server --region us-west1 --no-traffic
```

Notes: the ESP32-C3 needs stable Wi-Fi to reach the server; configure Google Cloud permissions (Artifact Registry and
Cloud Run) before deploying. The API has no authentication — anyone with the URL can change the lights.

### Key files

- **`app.py`** — Flask application logic (routes, in-memory state, QR code)
- **`templates/index.html`**, **`templates/base.html`** — web interface
- **`static/styles.css`** — page styling
- **`Dockerfile`** — container setup
- **`requirements.txt`** — pinned Python dependencies (Flask 3.1.0, Flask-Scss 0.5, qrcode 8.0, pillow 11.0.0, …)
- **`deployment_cmd.txt`** — the Cloud Run deployment commands above

## Documents

- [Final progress report — cover page](doc/report.png) ("LED Lights Controller via Web", December 2, 2024)
- Demo video: [static/media/resources/IMG_1993_optimized.mp4](static/media/resources/IMG_1993_optimized.mp4)
- Second iteration of the web UI (private): [xewe-labs/xewe-led-web-ui](https://github.com/xewe-labs/xewe-led-web-ui)
- Related: [XeWe LED OS](https://maxdokukin.com/projects/xewe-led-os) — ESP32 firmware for addressable LEDs with a local web interface ([xewe-labs/xewe-led-os](https://github.com/xewe-labs/xewe-led-os))
- Project page: [maxdokukin.com/projects/xewe-led-web-draft](https://maxdokukin.com/projects/xewe-led-web-draft)
