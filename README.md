<div align="center">

<br/>

```
 █████╗ ██╗   ██╗██████╗  █████╗ ██╗      ██████╗  ██████╗██╗  ██╗
██╔══██╗██║   ██║██╔══██╗██╔══██╗██║     ██╔═══██╗██╔════╝██║ ██╔╝
███████║██║   ██║██████╔╝███████║██║     ██║   ██║██║     █████╔╝ 
██╔══██║██║   ██║██╔══██╗██╔══██║██║     ██║   ██║██║     ██╔═██╗ 
██║  ██║╚██████╔╝██║  ██║██║  ██║███████╗╚██████╔╝╚██████╗██║  ██╗
╚═╝  ╚═╝ ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝ ╚═════╝  ╚═════╝╚═╝  ╚═╝
```

**Offline-first biometric authentication for the real world.**  
Face recognition + liveness detection — no internet required.

<br/>

![Platform](https://img.shields.io/badge/Platform-Android%20%7C%20iOS-black?style=for-the-badge&logo=android&logoColor=white)
![Kotlin](https://img.shields.io/badge/Kotlin-2.2.10-7F52FF?style=for-the-badge&logo=kotlin&logoColor=white)
![Swift](https://img.shields.io/badge/Swift-SwiftUI-F05138?style=for-the-badge&logo=swift&logoColor=white)
![TFLite](https://img.shields.io/badge/TensorFlow_Lite-2.16.1-FF6F00?style=for-the-badge&logo=tensorflow&logoColor=white)
![License](https://img.shields.io/badge/License-Proprietary-red?style=for-the-badge)

<br/>

</div>

---

## What is Auralock?

Auralock is a **secure, offline biometric verification system** built for environments where internet connectivity is unreliable or unavailable — remote worksites, field operations, and edge deployments.

It combines **real-time face detection**, **active liveness challenges** (blink, smile, head turn), and **AES-256 encrypted local storage** to deliver enterprise-grade identity verification entirely on-device.

> No cloud. No latency. No spoofing.

---

## Features

- 🔒 **Fully Offline** — SQLite-backed face embedding storage with zero cloud dependency
- 🧠 **AI-Powered** — MobileFaceNet ArcFace (128-dim embeddings) + SCRFD face detection
- 👁️ **Active Liveness Detection** — Anti-spoofing via EAR / MAR / Yaw challenge analysis
- 🛡️ **Passive Liveness** — Silent anti-spoofing via `silent_face_int8.tflite`
- 📐 **Quality Gating** — Lighting, centering, tilt, and bounding-box checks before any frame is accepted
- 🔐 **Encrypted Storage** — AES-256 biometric templates via Room + Crypto Security
- ☁️ **Optional Cloud Sync** — AWS Amplify mock sync queue for when connectivity is available
- 📱 **Cross-Platform** — Native Jetpack Compose (Android) + SwiftUI (iOS)

---

## Demo Flow

```
Camera Feed
    │
    ▼
┌─────────────────────────┐
│   FaceQualityChecker    │  ← Size · Center · Tilt · Lighting
└────────────┬────────────┘
             │ PASS
             ▼
┌─────────────────────────┐
│  ActiveLivenessDetector │  ← BLINK → SMILE → TURN_HEAD
└────────────┬────────────┘
             │ PASS
             ▼
┌─────────────────────────┐
│   FaceRecognitionEngine │  ← MobileFaceNet cosine similarity
└────────────┬────────────┘
             │
     ┌───────┴───────┐
     ▼               ▼
 ✅ GRANTED      ❌ DENIED
```

---

## Tech Stack

### Android

| Layer | Technology |
|---|---|
| Language | Kotlin 2.2.10 |
| UI | Jetpack Compose + Material 3 |
| Camera | CameraX 1.5.0 |
| ML Inference | TensorFlow Lite 2.16.1 |
| Architecture | MVVM + Clean Architecture |
| Async | Kotlin Coroutines + StateFlow |
| Local DB | Room 2.7.0 + AES-256 Crypto |
| Cloud Sync | AWS Amplify 2.14.11 (optional) |
| Build | Android Gradle Plugin 9.1.1 |

### iOS

| Layer | Technology |
|---|---|
| Language | Swift |
| UI | SwiftUI |
| Camera | AVFoundation (`AVCaptureSession`) |
| ML Inference | CoreML / Vision (native) |
| Architecture | MVVM + Combine |
| Local DB | WatermelonDB |
| Deployment Target | iOS 12.4+ |

---

## AI Models

All models live in `src/assets/models/` and run entirely on-device.

| Model | Purpose |
|---|---|
| `scrfd_500m_int8.tflite` | Face boundary detection (confidence: 0.95) |
| `mobilefacenet_arcface_int8.tflite` | 128-dim face embedding + cosine similarity |
| `silent_face_int8.tflite` | Passive liveness / anti-spoofing |

---

## Quality Rules

Before any liveness challenge begins, every frame is validated:

| Check | Rule |
|---|---|
| **Face Size** | Bounding box must be ≥ 150×150 px |
| **Centering** | Nose proxy within 20% of frame center |
| **Tilt** | Head roll must be < 0.26 rad (~15°) |
| **Lighting (Dark)** | Brightness score must be > 0.20 |
| **Lighting (Bright)** | Brightness score must be < 0.85 |

---

## Liveness Challenges

### Android — Sequential
```
IDLE  →  BLINK  →  SMILE  →  TURN_HEAD  →  COMPILING  →  RESULT
```

### iOS — Randomized
Each session picks a random challenge order with a **7-second timeout** per step.

### Thresholds

| Challenge | Metric | Android | iOS |
|---|---|---|---|
| Blink | EAR | < 0.24 | < 0.20 |
| Smile | MAR | > 0.40 | — |
| Head Turn | Yaw | < 0.70 or > 1.40 | > 20° or < -20° |

---

## Getting Started

### Prerequisites
- [Android Studio](https://developer.android.com/studio) (latest stable)
- A physical device or emulator running **Android 8.0+ (API 26)**
- Gemini API key (for AI Studio integration)

### Setup

```bash
# 1. Clone the repo
git clone https://github.com/shinchxn/Auralock.git
cd Auralock

# 2. Set up environment
cp .env.example .env
# Add your GEMINI_API_KEY to .env

# 3. Open in Android Studio
# File → Open → select this directory

# 4. Fix signing config
# Remove this line from app/build.gradle.kts:
# signingConfig = signingConfigs.getByName("debugConfig")

# 5. Run on device or emulator
```

### Required Permissions

**Android (`AndroidManifest.xml`)**
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

**iOS (`Info.plist`)**
```xml
<key>NSCameraUsageDescription</key>
<string>Auralock requires camera access for offline face recognition and liveness detection.</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Auralock requires location access to securely log the geolocation of biometric verifications.</string>
```

---

## Project Structure

```
Auralock/
├── app/                         # Android (Jetpack Compose)
│   └── src/main/java/com/example/
│       ├── viewmodel/           # BiometricViewModel.kt
│       ├── recognition/         # FaceQualityChecker.kt
│       ├── data/repository/     # BiometricRepository.kt
│       └── ui/views/            # LivenessCameraIconView.kt
├── ios/Auralock/                # iOS (SwiftUI)
│   ├── ActiveLivenessDetector.swift
│   └── FaceRecognitionEngine.swift
├── src/assets/models/           # TFLite model files
├── packages/edgeverify-module/  # Shared monorepo module
├── gradle/                      # libs.versions.toml
├── .build-outputs/              # CI/CD artifacts
└── docs/                        # Documentation
```

---

## Similarity Thresholds

| Platform | Threshold | Notes |
|---|---|---|
| Android | > 0.75 | Cosine distance comparison |
| iOS | > 0.65 | Mock deterministic matching |

---

## Testing

```bash
# Unit tests (JVM, no emulator needed)
./gradlew test

# Snapshot tests (Roborazzi)
./gradlew verifyRoborazziDebug
```

Test files under `app/src/test/java/com/example/`:
- `ExampleUnitTest.kt`
- `ExampleRobolectricTest.kt`
- `GreetingScreenshotTest.kt`

---

## Known Limitations

- EAR / MAR / Roll thresholds are hardcoded constants — not adaptive per user
- Tilt detection uses 2D sine approximation (no true 3D pose estimation)
- iOS `FaceRecognitionEngine` currently uses mock/deterministic matching (CoreML integration pending)
- No `LICENSE` or `CONTRIBUTING` file yet

---

## Roadmap

- [ ] Real CoreML model integration for iOS
- [ ] Adaptive liveness thresholds per device/lighting condition
- [ ] Admin dashboard for enrollment management
- [ ] Multi-face enrollment per identity
- [ ] True 3D head pose estimation
- [ ] Open source license

---

<div align="center">

Built with 🔐 for secure, offline-first identity verification.

</div>
