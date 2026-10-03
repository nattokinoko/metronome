# "Cadence Metronome" Web Application Development Specification

**Document Version**: v1.3.0

**Last Updated**: October 3, 2026

## 1. System Purpose

This system is a training Web application (Single File Component) designed to assist cyclists with maintaining and gradually improving their pedaling cadence (RPM: Revolutions Per Minute) on road bikes and indoor trainers.

To support seamless transitions from warm-up to target cadences (ramp-up / ramp-down), it incorporates a "Stepless Linear Interpolation Algorithm" that continuously adjusts click sound intervals over time, paired with a "Pre-rendered PCM Buffer Strategy" using the Web Audio API. This architecture provides an extremely precise training audio environment free from volume fluctuation, phase interference, or timing latency across all platforms (PC and mobile devices).

## 2. Version History

| Version | Revision Date | Summary of Revisions |
| ----- | ----- | ----- |
| **v1.0.0** | Sep 10, 2026 | Initial creation. Implemented continuous electronic beep playback with fixed BPM input and start/stop toggle control. |
| **v1.1.0** | Sep 28, 2026 | Feature expansion. Added input fields for Start Cadence, Target Cadence, and Transition Duration. Introduced real-time cadence adjustment via Stepless Linear Interpolation. |
| **v1.2.0** | Oct 1, 2026 | Precision enhancement. Adopted Look-ahead Scheduling anchored to `AudioContext.currentTime` to absorb JavaScript timer jitter (latency). |
| **v1.3.0** | Oct 3, 2026 | Audio engine overhaul. Replaced real-time oscillator and gain curve drawing with a Pre-rendered PCM Buffer Strategy (`AudioBuffer`) to eliminate periodic volume inconsistency and phase cancellation occurring at 128-sample boundaries. |

## 3. System Architecture & Environment

### 3.1 Operating Environment

* **Target Devices**: Smartphones (iOS / Android), PC, Tablets

* **Recommended Browsers**: Google Chrome, Apple Safari, Microsoft Edge, Mozilla Firefox (latest versions)

* **Deployment Form**: Standalone execution (Single File Component), PWA (Add to Home Screen) supported

* **Network Environment**: Full offline functionality available

### 3.2 Technology Stack

* **Frontend**: HTML5, CSS3 (CSS Variables, Flexbox/Grid)

* **Scripting Language**: JavaScript (ES6+ Vanilla JS / Single File)

* **Audio & Timing Control**: Web Audio API (`AudioContext`, `AudioBuffer`, `AudioBufferSourceNode`)

## 4. State Management & Data Model

As a single-file application, memory-resident state variables are managed dynamically in real time during execution:

| Variable / Object Name | Type | Initial Value | Description / Purpose |
| ----- | ----- | ----- | ----- |
| `startCadence` | `Number` | `70` | Initial cadence at session start (RPM) |
| `targetCadence` | `Number` | `90` | Final cadence reached and maintained after transition duration (RPM) |
| `transitionDurationSec` | `Number` | `300` | Cadence transition duration converted to seconds |
| `startTime` | `Number` | `0` | Baseline time (`AudioContext.currentTime`) recorded when pressing the Start button |
| `nextNoteTime` | `Number` | `0` | Scheduled timestamp (`AudioContext.currentTime`) for the next click sound |
| `isRunning` | `Boolean` | `false` | Execution status flag (`true`: Running, `false`: Stopped) |
| `audioCtx` | `AudioContext` | `null` | Master AudioContext object for the Web Audio API |
| `clickBuffer` | `AudioBuffer` | `null` | Pre-rendered PCM audio buffer containing uniform click sound data |

## 5. Functional Specifications

### 5.1 Input Controls

1. **Input Parameter Fields**:

   * **Start Cadence (**$C_{start}$**)**: Range `30` to `250` RPM (Default: `70`)

   * **Target Cadence (**$C_{target}$**)**: Range `30` to `250` RPM (Default: `90`)

   * **Transition Time (**$T_{trans}$**)**: Range `0` to `120` minutes (Default: `5` minutes, Step: `0.5` minutes)

2. **Validation & Fallback**:

   * Empty or invalid inputs automatically fallback to safe default values (70 RPM / 90 RPM / 0 min).

### 5.2 Cadence Transition Algorithm

1. **Initialization (Playback Start)**:

   * When the Start button is pressed ($t = 0$), the initial interval is calculated as $I_{start} = \frac{60}{C_{start}}$ [seconds], triggering the first click sound.

2. **Ramp Transition Phase (**$0 \le t < T_{trans}$**)**:

   * Based on elapsed time $t$ [seconds], the current target cadence $C(t)$ is calculated continuously via linear interpolation:

     $$
     C(t) = C_{start} + \left( \frac{C_{target} - C_{start}}{T_{trans}} \right) \cdot t
     $$

   * Upon each click sound, the subsequent click interval $I(t) = \frac{60}{C(t)}$ [seconds] is dynamically recalculated to update `nextNoteTime`.

3. **Hold Phase (**$t \ge T_{trans}$**)**:

   * Once elapsed time reaches $T_{trans}$, the cadence locks at $C(t) = C_{target}$, maintaining a constant tempo until manually stopped.

### 5.3 Audio Engine & Look-ahead Scheduler

1. **Pre-rendered PCM Buffer Strategy**:

   * Eliminates real-time envelope ramping calculations on oscillators. On initial execution, a 50ms PCM buffer (`AudioBuffer`) consisting of an 880Hz (A5 tone) sine wave with a 3ms linear attack and exponential decay is pre-baked and reused.

   * This completely eliminates volume drops and phase interference caused by Web Audio API 128-sample rendering block boundary aliasing (~2.7ms intervals).

2. **Look-ahead Scheduling**:

   * A JavaScript `setTimeout` loop running at ~25ms intervals schedules upcoming sounds up to `audioCtx.currentTime + 0.2` seconds (200ms look-ahead buffer) directly into the Web Audio API timeline.

   * Ensures zero timing jitter even during mobile background timer delays or browser rendering spikes.

### 5.4 User Interface & Controls

1. **Real-Time Cadence Display**:

   * Displays the current calculated cadence $C(t)$ rounded to the nearest integer (RPM) prominently in the center of the UI, synchronized with click beats.

   * Displays `--` when stopped.

2. **Toggle Control Button (Start / Stop)**:

   * Stopped State: Blue background (Label: "スタート" / "Start")

   * Running State: Red background (Label: "停止" / "Stop")

   * Tapping immediately halts the scheduler and safely resets audio timers and display states.

## 6. Non-functional Requirements & Technical Features

1. **100% Volume Consistency**:

   * Guarantees consistent playback volume on desktop and mobile platforms (iOS Safari / Android Chrome) by bypassing quantization and sampling boundary noise.

2. **Zero-Dependency Single-File Architecture**:

   * Operates completely offline without external JS frameworks (React, Vue, etc.) or audio assets (MP3, WAV).

3. **High-Contrast Responsive UI**:

   * Designed with standard dark mode to maintain readability under outdoor sunlight or indoor workout environments, featuring touch-friendly target dimensions.

```
eof

```
```

英語版の開発仕様書 `cadence_metronome_spec_en.md` を作成いたしました。

日本語版仕様書（v1.3.0）と完全に対応する形で構成しており、数式表記・PCMバッファ制御・ルックアヘッドスケジューリング等の技術用語も適切に英訳されています。