
## 🎛️ Hardware Mappings & Pinout

| Component | Driver / Interface | Pin / Address | Description |
| :--- | :--- | :--- | :--- |
| **User Button 1** | TCA9555 I2C Expander | Port 1 Pin 1 (`P1_1`) | **Volume UP (+15%)** |
| **User Button 2** | TCA9555 I2C Expander | Port 1 Pin 2 (`P1_2`) | **Volume DOWN (-15%)** |
| **User Button 3** | TCA9555 I2C Expander | Port 1 Pin 3 (`P1_3`) | **Mute / Unmute Toggle** |
| **BOOT Button** | Native ESP32-S3 GPIO | `GPIO 0` | **Manual Start / Stop Toggle** |
| **Audio DAC (Speaker)** | ES8311 | I2C `0x18`, I2S Port 1 | Audio playback & PA amplifier control |
| **Audio ADC (4-Mics)** | ES7210 | I2C `0x40`, I2S Port 0 | 4-Channel microphone array input |
| **RGB LED Strip** | WS2812 | `GPIO 38` | 7-LED status & visual volume bar |
| **I2C Bus** | ESP-IDF I2C Master | SDA: `GPIO 11`, SCL: `GPIO 10` | Codec & expander communication |

---

## 📁 Repository Structure

```text
ESP32-S3-with-Gemini-and-custom-persona/
├── backend/                        # Python FastAPI Relay Server
│   ├── server.py                   # FastAPI server & Gemini Live API WebSocket endpoint
│   ├── patient_persona.json        # Patient clinical record dataset (Samarth)
│   ├── requirements.txt            # Python dependencies (fastapi, uvicorn, google-genai, edge-tts)
│   ├── test_tts.py                 # Standalone TTS verification script
│   └── .env                        # Gemini API key & model configuration
├── main/                           # ESP32-S3 Firmware (ESP-IDF C Source)
│   ├── main.c                      # App entry point, FreeRTOS tasks, WebSocket client
│   ├── hardeware_driver/           # Codec drivers (bsp_board.c, ES8311, ES7210, TCA9555)
│   ├── rgb_led_driver/             # WS2812 RGB LED strip driver & volume level bar
│   ├── wifi_driver/                # Wi-Fi station mode driver (Power Save: DISABLED)
│   ├── CMakeLists.txt              # Main component CMake manifest
│   └── idf_component.yml           # ESP-IDF component dependencies
├── voice_jitter_technical_details.md   # Diagnostic analysis for audio jitter
├── SYSTEM_OPTIMIZATION_AND_FIXES_SUMMARY.md # System fixes & memory hardening summary
├── CMakeLists.txt                  # Top-level project CMake configuration
├── partitions.csv                  # Custom partition table
├── sdkconfig                       # ESP-IDF configuration manifest
└── README.md                       # Main project documentation
```

---

## 🚀 Quick Start Guide

### 1️⃣ Setting Up the Python Backend Server

1. Navigate to `backend/`:
   ```bash
   cd backend
   ```
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Set your Google Gemini API Key in `.env`:
   ```env
   GEMINI_API_KEY=your_actual_gemini_api_key_here
   GEMINI_LIVE_MODEL=gemini-3.1-flash-live
   PORT=8008
   ```
4. Start the server:
   ```bash
   python server.py
   ```

#### Local Redis cache and duplicate-name confirmation

The backend uses Redis for the API 1 roster (24-hour recovery TTL), API 2 personas (5-minute TTL), and temporary per-WebSocket patient selection state. Configure `REDIS_URL` in `backend/.env`; set `REDIS_TLS=true` for a TLS endpoint such as Upstash. Keep credentials in the ignored `backend/.env`, never in source files or logs. The ESP32 connects only to the FastAPI WebSocket.

When API 1 returns more than one patient for a name, the assistant asks for the last four phone digits. The backend checks those digits against only the pending candidates in the cached roster; it never sends phone numbers to Gemini. A unique match is required before API 2 is called. Pending choices expire after three minutes. If Redis is temporarily unavailable, the server logs a warning and falls back to its in-memory cache for that process.

If API 1 temporarily reports zero patients, seed the complete JSON response you saved from API 1 into Redis (from `backend/`, with Redis running):

```powershell
python -m pip install -r requirements.txt
python seed_patient_roster_redis.py C:\path\to\api1_roster.json --doctor-id 673334d939a1270181963600
```

The importer checks that the response's `total` equals the number of parsed records and verifies the Redis write. It will not overwrite an existing roster unless `--replace` is added. The patient list stays in Redis rather than in the source repository. An empty API response will not replace the cached full roster.

---

### 2️⃣ Building and Flashing the ESP32-S3 Firmware

1. Open an ESP-IDF terminal (v5.5).
2. The current firmware connects to `ws://192.168.50.53:8008/ws/live/673334d939a1270181963600`. This is the backend PC's LAN address; the ESP32 and PC must be on the same network. If the PC's address changes, update **Relay Server Host / IP / Domain** in `idf.py menuconfig` and rebuild. API 1 and API 2 separately use `http://200.97.162.162:8000`.
3. Build, flash, and open serial monitor:
   ```bash
   idf.py -p COM6 flash monitor
   ```

---

## 🎙️ Spoken Voice Commands & Interactivity

### Sample Questions You Can Ask:
- **"Hello Assistant"** $\rightarrow$ *"Hello Samarth! I am right here. What would you like to check today?"*
- **"What is my weight?"** $\rightarrow$ *"Hi Samarth! You currently weigh 70 kg, which reflects a weight loss of 16.2 kg."*
- **"Who is my doctor?"** $\rightarrow$ *"Your assigned doctor is Dr. Samarth Gupta."*
- **"What is my HbA1c?"** $\rightarrow$ *"Your latest HbA1c level is 5.52%."*

---

## 📜 License

Distributed under the MIT License. See `LICENSE` for more information.
