# Claude handoff: patient persona lookup returns 404

Please inspect the attached source bundle and identify a safe code or API-contract fix. Do not assume that the sample persona ID belongs to the selected patient, and do not return one patient's persona for another patient.

## Expected flow

1. ESP32 keeps a WebSocket open to the local FastAPI relay at `/ws/live/{doctor_id}`.
2. Gemini Live recognizes the doctor's request and calls the backend patient lookup tool.
3. Backend finds the patient in the cached API 1 roster (`/agent/patients/{doctor_id}`), which is persisted in Redis.
4. Backend passes that patient's API 2-compatible identifier to `/persona/{patient_id}` with the configured bearer token, caches the persona, and sends the result to Gemini for speech.
5. Duplicate names are disambiguated by the final four phone digits.

## Current finding

- The WebSocket and Gemini Live session start successfully. The ESP32 receives complete-turn events.
- API 1 currently returns an empty roster for this doctor, so the backend uses the cached 165-row roster supplied earlier.
- The cached roster rows inspected contain only `id`, `first_name`, `last_name`, and `mobile`; they do not contain `persona_id` or another explicit API 2 mapping field.
- For the selected roster IDs, API 2 returns HTTP 404. A read-only check using the configured token also returned 404 for both selected IDs and HTTP 200 for the known sample ID `6a0185bd7aeac73c4f44804a`.
- This points to an API 1/API 2 identifier or data-assignment mismatch. The backend already calls `/persona/{roster_id}` as configured. Please verify whether API 1's `id` is actually the key API 2 expects, and whether an ID-translation endpoint or explicit `persona_id` field is required.
- Do not hard-code the sample ID or silently use another patient's persona. Return a clear recoverable error if no verified mapping exists.
- An earlier fuzzy-name match selected “Lisha Karar” after the doctor asked for “Esha Karar.” Fuzzy matching has been removed from `backend/server.py`; unknown names should trigger a clarification instead of selecting a similar patient.
- Firmware logs also show playback-ring overflow warnings. These may clip speech but are separate from the API 2 404.

## Relevant code

- `backend/server.py`: API 1 roster caching, name/ID/phone matching, API 2 persona fetching, Redis cache, Gemini tools, and WebSocket relay.
- `backend/seed_patient_roster_redis.py`: imports a validated full roster into Redis.
- `backend/requirements.txt`, `backend/.env.example`: backend dependencies and configuration names.
- `main/main.c`, `main/wifi_driver/`, `main/rgb_led_driver/`: ESP32 microphone, audio playback, Wi-Fi, and WebSocket client.
- `README.md`, `PROJECT_SYSTEM_ARCHITECTURE_DOCUMENTATION.md`: project architecture and setup.
- `backend/.env` in this handoff is redacted. Use local credentials only for a controlled API test; never commit or publish live credentials.

## Please answer

1. Is the current API 1 `id` suitable for `/persona/{id}`? What exact field or API call should provide the persona ID?
2. If a translation is needed, propose a safe doctor-scoped mapping/cache design with explicit handling for missing, ambiguous, and stale mappings.
3. Review the name matching and duplicate phone last-four logic for accidental cross-patient matches.
4. Identify only changes that can be implemented in this repository. Clearly separate code defects from API data/contract changes that require the API owner.
5. Preserve the persistent WebSocket/Gemini session; keep patient and persona data scoped by doctor and patient ID.

Do not print or repeat any credentials. The bundle intentionally contains no live API key, bearer token, Redis URL/password, patient persona JSON, build output, or virtual environment.
