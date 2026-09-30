# ASTRAQ — System Architecture

## 1. Overview

ASTRAQ is built as a modular simulation system.

The main simulation can run inside the browser. An optional Python server provides a second engine, REST APIs, WebSocket telemetry, batch testing, experiments, and server-side video analysis.

The system is divided into four main parts:

```text
User Interface
     ↓
Engine Provider
     ↓
Simulation Engine
     ↓
Tracking Loop
     ↓
Gimbal
     ↓
Camera Image
     └──────────────→ Tracking Loop
```

## 2. Main components

```text
ASTRAQ
│
├── web/
│   ├── React + TypeScript UI
│   ├── 3D scene
│   ├── simulation core
│   ├── Web Workers
│   ├── state management
│   └── telemetry providers
│
├── server/
│   ├── FastAPI application
│   ├── Python simulation engine
│   ├── WebSocket telemetry
│   ├── experiments
│   ├── batch runs
│   └── video benchmark
│
├── desktop/
│   └── Electron desktop wrapper
│
└── docs/
    └── project documentation
```

## 3. Browser architecture

The browser is the default runtime.

```text
                    Browser
┌───────────────────────────────────────────────┐
│                                               │
│  React UI                                     │
│      ↓                                        │
│  Zustand state                                │
│      ↓                                        │
│  TelemetryProvider                            │
│      ↓                                        │
│  Web Worker                                   │
│      ↓                                        │
│  SimulationEngine                             │
│                                               │
│  3D Scene ← live snapshots / image / history │
│                                               │
└───────────────────────────────────────────────┘
```

The simulation engine runs in a Web Worker so that the main interface remains responsive.

The 3D scene displays simulation data. It does not calculate the tracking algorithm.

## 4. Engine providers

ASTRAQ uses a provider design so the UI does not need to know where the simulation data comes from.

### Local provider

Runs the TypeScript simulation inside the browser.

```text
Browser
  ↓
LocalEngineProvider
  ↓
Web Worker
  ↓
TypeScript engine
```

This is the default mode.

### Remote provider

Connects the UI to the Python FastAPI server.

```text
Browser
  ↓ WebSocket
FastAPI
  ↓
Python simulation engine
```

### Replay provider

Reads a saved recording and plays it back through the same UI.

## 5. Simulation loop

The main camera loop runs at 30 Hz.

The control loop can run at 60 Hz.

```text
1. Target moves
       ↓
2. Disturbances are applied
       ↓
3. Camera renders the current image
       ↓
4. Detector finds beacon candidates
       ↓
5. Pixel position is converted to direction
       ↓
6. Kalman filter estimates position and rate
       ↓
7. PID controller calculates gimbal rate
       ↓
8. Gimbal moves
       ↓
9. New camera image is produced
```

The loop continues until the simulation is stopped.

## 6. Core modules

| Module | Purpose |
|---|---|
| `config.ts` | Simulation settings and defaults |
| `geometry.ts` | Directions, angles, coordinate conversion, and orbit geometry |
| `target/` | Target movement and trajectories |
| `optics/` | Camera and sensor calculations |
| `detection/` | Beacon detection and AI verifier |
| `estimation/` | Kalman filter |
| `control/` | PID controller, gimbal, and search movement |
| `tracking/` | Tracking state machine |
| `disturbance/` | Noise, jitter, weather, motion, and other disturbances |
| `analysis/` | Metrics, link values, and performance results |
| `telemetry/` | Snapshot, recording, CSV, and JSON data |
| `video/` | Video benchmark |
| `demo/` | Guided demo sequence |
| `engine.ts` | Connects the full simulation loop |

## 7. Detection flow

```text
Camera frame
    ↓
Noise cleanup
    ↓
Background estimate
    ↓
Bright candidate regions
    ↓
Candidate checks
    ↓
Kalman prediction gate
    ↓
AI beacon verifier
    ↓
Best valid candidate
    ↓
Sub-pixel centroid
```

The detector works only with the camera image and other observable data.

## 8. Estimation flow

The Kalman filter works on azimuth and elevation rather than directly on pixels.

```text
Pixel measurement
      ↓
Camera geometry
      ↓
Azimuth / elevation
      ↓
Kalman prediction
      ↓
Estimated position + velocity
```

This makes the estimate less dependent on gimbal movement.

## 9. Control flow

```text
Target estimate
      ↓
Angular error
      ↓
PID
  ├── P
  ├── I
  ├── D
  └── Kalman feed-forward
      ↓
Gimbal rate command
      ↓
Rate / acceleration limits
      ↓
Gimbal movement
```

## 10. State machine

The tracking supervisor manages the tracking state.

```text
SEARCHING
   ↓
DETECTED
   ↓
ACQUIRING
   ↓
TRACKING
   ↓
LOCKED
   │
   └── target lost
          ↓
        LOST
          ↓
     REACQUIRING
          ↓
       TRACKING
```

The state machine also records why a state change happened.

Examples:

- Detection confirmed
- Lock condition reached
- Too many missed frames
- Beacon visible again
- Reacquisition completed

## 11. Ground truth separation

Ground truth is kept separate from the normal tracking path.

The tracking system uses observable simulation data.

Ground truth is used by the metrics system to measure the result.

This prevents the tracker from using the correct answer directly.

## 12. Frontend data flow

The engine sends:

- Simulation snapshots
- Sensor images
- Configuration updates
- Status messages
- Errors

The frontend stores high-rate data in buffers and sends a lower-rate copy to the visible interface.

This helps keep the 3D scene and charts smooth.

## 13. Python server architecture

```text
FastAPI
│
├── REST API
│
├── WebSocket
│
├── SimulationService
│       ↓
│   Python engine
│
├── Experiment service
│
├── Batch runner
│
└── Video analysis
```

The server can:

- Start and stop the simulation
- Change configuration
- Send telemetry
- Record experiments
- Run batch tests
- Analyse videos
- Generate reports

## 14. Communication

The UI can send commands such as:

```text
start
pause
reset
config
replaceConfig
demo
mode
manual
reacquire
timeScale
imageRate
```

The engine can return:

```text
snapshot
config
status
error
```

The WebSocket can also carry sensor image frames.

## 15. Desktop architecture

The desktop application uses Electron.

```text
Electron
   ↓
Local HTTP server
   ↓
Production web build
   ↓
Browser UI
   ↓
Browser simulation engine
```

The desktop package can run without the Python server.

## 16. Design rules

ASTRAQ follows these main design rules:

1. Keep simulation logic separate from the UI.
2. Keep 3D rendering separate from tracking logic.
3. Keep backend communication behind a provider interface.
4. Keep ground truth away from the normal tracking path.
5. Keep the TypeScript and Python telemetry format aligned.
6. Keep individual simulation modules small and replaceable.
7. Keep testing independent from the visual interface.

## 17. Why this architecture is useful

This structure makes it easier to:

- Run the main demo without a server.
- Test the engine without the 3D interface.
- Replace the detector later.
- Connect a real telemetry source later.
- Run many simulations for testing.
- Compare browser and Python results.
- Add hardware interfaces without rewriting the complete UI.
