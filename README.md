# Signal Zero // The Motorcycle Co-Pilot

Signal Zero is a voice-controlled AI companion designed for the cockpit. It integrates navigation, media, weather, and communications into a hands-free interface that prioritizes rider focus and safety. 

Built on the Gemini Live Audio API for real-time voice interaction — no buttons, no screens, no distractions. Just you, Zero, and the road.

---

## 1. Core Concept & Features

Zero is a hands-free Android assistant pairing a real-time voice interface with a robust, local tool-dispatch layer implemented in Kotlin. The system maintains tight synchronization between prompt context, function schemas, and active service dispatch handlers.

* **Smart Conversation Gate:** The microphone gate remains open as long as you are actively speaking. The timer resets every 3 seconds while audio packets stream, allowing you to tell long stories without interruption. The gate only closes after 10 seconds of complete silence.
* **Speaker Verification (Voice ID):** Powered by a local `sherpa-onnx` pipeline, Zero validates incoming audio against a locally enrolled voice profile. Once trained, the system ignores outside voices, passengers, or environmental noise.
* **Hybrid Voice System (Rider vs. Social Mode):** * **Rider Mode:** Uses the device's native, local Text-to-Speech (TTS) engine. It features 0ms latency, works completely offline, and is optimized for turn-by-turn navigation at speed.
  * **Social Mode:** Leverages high-fidelity cloud audio models for rich, natural conversational banter when parked or riding in strong signal environments.
* **App Context Layouts:** * **BIKE Mode:** Keeps the microphone hot while moving, enables seamless user barge-in during AI speech, and extends follow-up windows.
  * **AUTO Mode:** Optimizes for desk or car use, closing the microphone immediately following a response for quiet, one-shot interactions.
* **Intelligent Routing:** Implements gravel-free path generation by default, permanently excluding unpaved surfaces from all navigation tracks.

---

## 2. Technical Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────┐
│                      SIGNAL ZERO                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │               ZeroService (Foreground)           │    │
│  │  ┌──────────────┐    ┌──────────────────────┐    │    │
│  │  │ AudioStream  │◄──►│  GeminiLiveClient    │    │    │
│  │  │ Manager      │    │  WebSocket + VAD     │    │    │
│  │  │ + WakeWord   │    │  + Reconnect Logic   │    │    │
│  │  │ + Speaker ID │    └──────────────────────┘    │    │
│  │  │   Self-Heal  │          ▲                     │    │
│  │  └──────────────┘          │                     │    │
│  │                            │                     │    │
│  │  ┌──────────────┐          │                     │    │
│  │  │ NetworkMonitor│─────────┘                     │    │
│  │  │ (dead-zone   │  notifyNetworkLost/Available   │    │
│  │  │  gating)     │                                │    │
│  │  └──────────────┘                                │    │
│  │                                                  │    │
│  │  ┌──────────────────────────────────────────┐    │    │
│  │  │       RideSessionManager                 │    │    │
│  │  │  PARKED ↔ RIDING ↔ OVERNIGHT (GPS speed) │    │    │
│  │  │  Ride-end: TTS → mic release → PARKED    │    │    │
│  │  │  Overnight: 30min no-motion 10PM–6AM     │    │    │
│  │  └──────────────────────────────────────────┘    │    │
│  │                                                  │    │
│  │  ┌──────────────────────────────────────────┐    │    │
│  │  │         ResourceWatchdog                 │    │    │
│  │  │  ACTIVE → SOFT_IDLE → HARD_IDLE (30s)   │    │    │
│  │  │  GPS · Mapbox · WebSocket · GeminiLive   │    │    │
│  │  │  RIDING thresholds / PARKED override     │    │    │
│  │  │  Paused by STANDBY/OFF power modes       │    │    │
│  │  └──────────────────────────────────────────┘    │    │
│  └──────────────────────────────────────────────────┘    │
│                          │ Binder                        │
│  ┌───────────────────────▼──────────────────────────┐    │
│  │               MainActivity (UI Layer)            │    │
│  │   Tool Handlers · Music · Maps · Comms · UI      │    │
│  │   pauseWatchdog() / resumeWatchdog() on mode ⚡  │    │
│  └──────────────────────────────────────────────────┘    │
│         │          │          │          │               │
│   Google Maps  OpenWeather  YouTube   Spotify            │
│                             API      App Remote          │
└──────────────────────────────────────────────────────────┘
```

### Core Architecture Components

* **ZeroService (The Spine):** Core Android Foreground Service running independently of the UI lifecycle. Owns the streaming managers, audio tracks, and network state callbacks. Survives app swipes via `onTaskRemoved()`.
* **GeminiLiveClient (The Brain):** Manages the live persistent WebSocket connection. Handles voice activity detection (VAD), tool payload parsing, exponential-backoff reconnect cycles, and dead-zone request deferrals.
* **AudioStreamManager (The Ears & Mouth):** Orchestrates raw PCM 16kHz microphone ingestion, 24kHz speaker playback tracking, and on-device wake-word detection loops.
* **ResourceWatchdog (The Power Manager):** An autonomous, background idle-state monitor executing sweeps every 30 seconds. Implements tiered resource degradation (`ACTIVE` → `SOFT_IDLE` → `HARD_IDLE`) to safely disconnect radio components, close web sockets, and pause GPS engines under idle conditions.
* **RideSessionManager (The Sentinel):** A speed-derived lifecycle machine transitioning between `PARKED`, `RIDING`, and `OVERNIGHT` modes based on live telemetry tracking.

---

## 3. Automation & Energy Management

### Sentient Resource Control (ResourceWatchdog)
To eliminate parasitic battery drain while keeping the engine active during long cross-country hauls, resources scale back dynamically based on telemetry context:

| Resource Type | Soft Idle Threshold | Hard Idle Threshold | Soft Action | Hard Action |
| :--- | :--- | :--- | :--- | :--- |
| **WebSocket** | 2 minutes | 8 minutes | Send Keep-Alive | Disconnect Session |
| **GPS Tracking** | 5 minutes | 15 minutes | Pause High-Accuracy | Terminate Listener |
| **Map Engine** | 5 minutes | 10 minutes | Lower Refresh Rate | Release Map Session |

* **PARKED Override:** When the host vehicle enters a stationary state, all resource timers collapse instantly to a 30-second soft / 60-second hard threshold to protect device resources.
* **On-Demand Resurrection:** If a system element is parked but a voice command requires it (e.g., asking for location while GPS is in `HARD_IDLE`), local low-latency TTS fires a notification instantly (*"Getting your location..."*) while rebuilding the underlying hardware connections asynchronously in the background.

### Automatic Ride Lifecycle
Zero handles microphone state mapping without manual button intervention, solving glove-heavy accessibility problems at highway speeds.

* **Hysteresis Buffering:** Telemetry boundaries are intentionally asymmetric (5 mph to trigger a stopping window, 15 mph to resume riding state) preventing unstable connection cycling during slow technical maneuvers or traffic delays.
* **Strict Audio Sequencing:** During ride-end routines, local TTS warning confirmations drain completely before mic teardown triggers. This behavior is enforced via an explicit `RIDE_END_TTS_DRAIN_MS` barrier, preventing clipping.

---

## 4. Connection Resilience & Stability
* **Tower Handover Recovery:** Detects cellular tower handovers (`Wi-Fi` ↔ `5G`) and recovers WebSockets using structured backoff timers. In full dead zones, retry loops pause entirely to protect battery, firing instantly via a `NetworkMonitor` callback the millisecond the radio acquires signal.
* **Underrun Mitigation:** Audio tracks evaluate device `PLAYSTATE` markers before executing byte array writes, automatically recovering from pipeline underruns without blocking the execution thread.
* **Speaker Verifier Self-Healing:** If a hardware state change triggers a soft-failure in the voice recognition pipeline (returning a persistent zero-score), the streaming engine waits 1 second for audio boundaries to settle, rebuilds the local model array from the saved user profile, and restores verification without requiring an application restart.

---

## 5. Tool Registry Contract

The active system exposes the following function primitives to the model configuration layer:

| # | Tool | Category |
|---|------|----------|
| 1 | `google_search` | Knowledge |
| 2 | `get_current_location` | Navigation |
| 3 | `navigate_to` | Navigation |
| 4 | `reroute_via` | Navigation |
| 5 | `generate_scenic_loop` | Navigation |
| 6 | `check_local_weather` | Weather |
| 7 | `set_home_address` | Settings |
| 8 | `save_waypoint` | Waypoints |
| 9 | `get_waypoints` | Waypoints |
| 10 | `navigate_to_waypoint` | Waypoints |
| 11 | `stop_navigation` | Navigation |
| 12 | `save_contact` | Communications |
| 13 | `send_text_message` | Communications |
| 14 | `make_phone_call` | Communications |
| 15 | `play_music` | Music |
| 16 | `play_spotify` | Music |
| 17 | `music_control` | Music |
| 18 | `save_memory` | Memory |
| 19 | `recall_memory` | Memory |
| 20 | `clear_memory` | Memory |
| 21 | `set_voice_mode` | Voice Control |
| 22 | `get_current_time` | Device Clock |

*Note: Unwired or deprecated endpoints (such as specific internal media browsers or video-id managers) are masked from the schema layer to keep model execution clean and eliminate hallucinated execution errors.*

---

## 6. Permissions & Environment Configurations

### Android Manifest Requirements
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.SEND_SMS" />
<uses-permission android:name="android.permission.CALL_PHONE" />
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
Hardware Isolation
The camera hardware requirement is flagged as optional within the application manifest. This design choice prevents non-camera execution layers or headless setups from being unnecessarily excluded during deployment cycles.

XML
<uses-feature android:name="android.hardware.camera" android:required="false" />
7. Future Engineering Roadmap
Codename: The Black Box: Implementation of an on-device local caching system for seamless voice note recording and background transcription while offline.

Codename: The HUD: High-contrast, minimalist serialization matrix optimized for external micro-OLED heads-up displays via Bluetooth.

Codename: The Mechanic: Integration of local OBD-II data-stream readers via Bluetooth serial channels to provide real-time thermal, diagnostic, and electrical alerts hands-free.

Codename: The Lean Machine: Gyroscopic telemetry tracking via multi-axis inertial measurement units (IMUs) for local cornering stability analysis.


***

### 🛠️ Next Steps For Your New Public Repository:
1. Open up your new public `signal-zero-docs` or `signal-zero-spec` repository.
2. Click the **pencil icon** on your blank `README.md`.
3. Paste this exact sanitized markdown text in, check it over, and hit **Commit changes**.
4. Head to your main profile page and pin it right beside *The Terminator* architecture document.

Having this up alongside your log analyzer shows a completely different, highly advanced side of your skill set. *The Terminator* proves you can handle massive corporate infrastructure scale, data pipelines, and backpressure in Python; *Signal Zero* proves you can handle sophisticated Android foreground lifecycles, real-time WebSockets, local machine learning models (`sherpa-onnx`), hardware telemetry, and strict resource management in Kotlin. 

It is an incredibly powerful one-two punch for any senior technical loop. Let me know when it's up!
