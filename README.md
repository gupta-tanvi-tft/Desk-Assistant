# ESP32-S3 Gemini Voice Assistant

This project connects a Waveshare ESP32-S3-AUDIO board to a Python FastAPI relay, Gemini Live, and the patient roster/persona APIs. The firmware captures and plays audio; the backend handles Gemini sessions, patient matching, API calls, Redis caching, and WebSocket audio transport.

## Quick start

### Backend (Windows PowerShell)

```powershell
cd C:\Users\tanvi-admin\Downloads\Desk_Assistant\gemini_voice_assistant_fixed\backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
notepad .env
python server.py
```

Set a valid `GEMINI_API_KEY` and the current `PERSONA_API_TOKEN` in the private `backend/.env`. Configure Redis if using a remote Redis service. The backend listens on port `8008` by default; `http://127.0.0.1:8008/` is its basic health endpoint.

### Firmware

Use an ESP-IDF 5.5 terminal. In the project root, set the ESP32-S3 target, check Wi-Fi and relay settings in `idf.py menuconfig`, then build, flash, and monitor:

```powershell
cd C:\Users\tanvi-admin\Downloads\Desk_Assistant\gemini_voice_assistant_fixed
idf.py set-target esp32s3
idf.py menuconfig
idf.py -p COM6 flash monitor
```

The current WebSocket relay is `ws://192.168.50.53:8008/ws/live/673334d939a1270181963600`. The ESP32 must be able to reach the PC running the backend over the LAN. API 1 and API 2 use the separate host `http://200.97.162.162:8000`.

## Patient lookup behavior

The backend matches a spoken name in API 1's doctor roster. If multiple patients have the same full name, it asks for the last four phone digits and checks them only against those matching roster rows. It then fetches the selected patient's persona from API 2 using that patient's ID. Redis caches the roster, personas, ID mappings, and short-lived conversation state.

## Documentation

See [Architecture and Data Flow](./ARCHITECTURE_AND_DATA_FLOW.md) for the detailed system diagram, conversation pipeline, patient matching rules, Redis behavior, environment variables, logs, flashing steps, and troubleshooting guidance.

## Keep credentials and patient data private

Do not commit `backend/.env`, API keys, bearer tokens, Wi-Fi passwords, roster exports, or patient records. The generated `sdkconfig` can contain local Wi-Fi settings; review it before sharing. Treat backend logs as sensitive because they may include speech transcripts, patient names, and patient IDs.
