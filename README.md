# Auralock - Secure Offline Biometric Detection

## SECTION 1: PROJECT OVERVIEW
Auralock is an offline-first face recognition and liveness detection application designed for both Android and iOS devices. The application features real-time facial tracking, telemetry evaluation, interactive liveness tests (blinking, smiling, turning head), and offline SQLite-based facial embedding verification. The Android application requires a minimum SDK of 26 (Android 8.0) and a target SDK of 36, utilizing CameraX for video preview and Jetpack Compose for its UI. The iOS application targets iOS 12.4+ and utilizes SwiftUI and Combine for its architecture.

## SECTION 2: TECH STACK
**Android**
* **Language & Version:** Kotlin 2.2.10
* **Build System & Version:** Gradle SDK via Android Gradle Plugin 9.1.1
* **Inference Runtime & Version:** TensorFlow Lite 2.16.1 (`org.tensorflow:tensorflow-lite:2.16.1`)
* **Camera API:** CameraX (`androidx.camera:camera-camera2`, `camera-lifecycle`, `camera-view`, `camera-core` v1.5.0)
* **Architecture Components:** ViewModel (`AndroidViewModel`), Kotlin Coroutines, StateFlow
* **Dependency Injection:** None (uses standard constructor injection and local variables)
* **Async Pattern:** Kotlin Coroutines (`kotlinx-coroutines-android` v1.10.2)
* **UI Approach:** Jetpack Compose (`androidx.compose` BOM 2024.09.00, Material 3)
* **Other Notable Libraries:** Room 2.7.0 (with KTX and Crypto security), AWS Amplify 2.14.11 (API, DataStore, Auth Cognito), Moshi, Retrofit, OkHttp.

**iOS**
* **Language & Version:** Swift (SwiftUI)
* **Deployment Target:** iOS 12.4
* **Inference Runtime:** Native Mock Wrappers (intended for CoreML / Vision).
* **Camera API:** `AVFoundation` (`AVCaptureSession` wrapped in `CameraManager`)
* **Architecture Pattern:** MVVM (`ObservableObject` with `@Published` properties)
* **Reactive Framework:** Combine
* **UI Approach:** SwiftUI
* **Other Notable Libraries:** WatermelonDB (integrated via Podfile custom setup), React Native bridges.

## SECTION 3: REPOSITORY STRUCTURE
* **.build-outputs:** CI/CD build outputs.
* **android:** Original React Native Android deployment folder.
* **app:** Pure Jetpack Compose Android application codebase.
* **docs:** Project documentation.
* **gradle:** Gradle configurations and `libs.versions.toml` library catalog.
* **ios:** Native iOS Swift and SwiftUI source code and Podfiles.
* **packages:** Monorepo package dependencies.
* **scripts:** Application build and transformation scripts.
* **src:** React Native components and shared asset directories (`src/assets/models/`).

## SECTION 4: ARCHITECTURE
**Android**
The Android component follows an MVVM architecture using Clean Architecture principles locally. The UI layer communicates with `BiometricViewModel.kt` (extending `AndroidViewModel`). `BiometricViewModel` exposes `StateFlow` streams (`verificationState`, `livenessStep`, `enrolledFaces`) that Jetpack Compose UI elements observe via `collectAsStateWithLifecycle()`. Data logic is handled via a dedicated `BiometricRepository.kt` class which interacts with a Room Database (`AppDatabase.kt`) holding entities (`EnrolledFace`, `AuthLog`, `SyncQueueItem`). Telemetry checks are processed synchronously within the main logic flow without third-party DI frameworks like Hilt.

**iOS**
The iOS component uses an MVVM architecture via `Combine` protocols. The main screen (`VerifyScreen`) interacts with `BiometricViewModel.swift`, which implements `ObservableObject`. Operations such as `FaceRecognitionEngine` and `ActiveLivenessDetector` map directly onto `@Published` string states (`verificationStateText`) which automatically trigger UI recomposition in SwiftUI.

## SECTION 5: KEY CLASSES AND FILES
* **`app/src/main/java/com/example/viewmodel/BiometricViewModel.kt`**: `BiometricViewModel` - Orchestrates the entire liveness tracking state machine, executes telemetry evaluations, controls Room DB offline matching, and manages AWS cloud mock sync routines.
* **`app/src/main/java/com/example/recognition/FaceQualityChecker.kt`**: `FaceQualityChecker` - Verifies lighting conditions, face positioning, minimum bounding box areas, and face pitch/yaw angles before accepting a frame.
* **`app/src/main/java/com/example/ui/views/LivenessCameraIconView.kt`**: `LivenessCameraIconView` - Handles the animated camera overlay with spinning rings, blinking eyes, smiles, and directional arrows based on the interactive liveness state.
* **`app/src/main/java/com/example/data/repository/BiometricRepository.kt`**: `BiometricRepository` - Manages persistence for SQL logs, enrollment seeds, simulation of AWS syncing, and AES-256 encrypted storage of biometric templates.
* **`ios/Auralock/ActiveLivenessDetector.swift`**: `ActiveLivenessDetector` - Maintains a 30-frame sequence analyzing mock `EAR`, `MAR`, and generic offsets against a 7-second timeout for interactive iOS challenges.
* **`ios/Auralock/FaceRecognitionEngine.swift`**: `FaceRecognitionEngine` - Mock class validating facial similarities using dummy tensor dimensions and deterministic matching mechanisms via `ComputeSimilarity`.

## SECTION 6: FACE DETECTION AND QUALITY RULES
The codebase enforces strict quality checks on captured face bounding boxes (`FaceQualityChecker.kt` / `FaceQualityChecker.swift`):
* **Face Setup Constraint:** SCRFD detection mock returns a confidence of `0.95f`.
* **Face Size Rule:** The minimum allowed bounding box area is strictly checked to be `< 22500.0` pixels (implying a minimum size of 150x150 relative to the image).
* **Centering Constraint:** The face's nose tip proxy (derived from the box center) must rest within `0.20` (20%) of the total frame width and height relative to the absolute center.
* **Tilt Threshold:** The absolute roll angle of the head (approximated from eye coordinates) cannot exceed `0.26 radians` (~15 degrees).
* **Lighting Rule:** Pixel brightness scores below `0.20` flag as "TOO_DARK", while scores above `0.85` flag as "HARSH_SUNLIGHT".

## SECTION 7: LIVENESS FLOW
The application executes an Interactive Liveness Flow using calculated offsets (EAR, MAR, YAW):
* **Challenges Present:** Blink (`BLINK` or `.blink`), Smile (`SMILE` or `.smile`), Head Turn (`TURN_HEAD` or `.turnLeft` / `.turnRight`).
* **Progression Type:**
  * **Android:** Sequential logic steps: `IDLE` -> `BLINK` -> `SMILE` -> `TURN_HEAD` -> `COMPILING`.
  * **iOS:** Random challenge selector (`ChallengeType.allCases.filter { $0 != previous }.randomElement()`).
* **Timeouts:** iOS enforces a strict `7.0` second timeframe (`challengeStartTime`) before resetting.
* **Thresholds:**
  * Blink detection evaluates Eye Aspect Ratios (EAR) mapped below `0.24` (Android) or `0.2` (iOS).
  * Smile detection assesses Mouth Aspect Ratios (MAR) mapping above `0.40` (Android).
  * Head Turn registers yaw orientations falling below `0.70` or jumping above `1.40` (Android), corresponding to explicit mock checks (`> 20 / < -20`) in iOS.

## SECTION 8: PIPELINE STATUS VALUES
**VerificationState (Android)**
* `Idle`: The starting point where no evaluation is actively compiling.
* `InteractiveRunning`: The system is tracking a designated challenge requirement (matches a `String` message).
* `Processing`: Telemetry passed, compiling identity checks (`Processing("Identity verified")`).
* `Result`: The final calculation mapping trust factors alongside `success`, `title`, `details`, `trustScore`, and explicit liveness/similarity properties.

## SECTION 9: ANDROID SETUP
To build and configure the Android source files locally, review the `app/build.gradle.kts` configuration logic.
Ensure the `models` files (`mobilefacenet_arcface_int8.tflite`, `scrfd_500m_int8.tflite`, `silent_face_int8.tflite`) reside properly within the application scopes.
**Permissions required in `AndroidManifest.xml`:**
* `android.permission.CAMERA`
* `android.permission.INTERNET`
* `android.permission.ACCESS_NETWORK_STATE`
* `android.permission.USE_BIOMETRIC`
* `android.permission.ACCESS_FINE_LOCATION`
**Versions:** Kotlin 2.2.10, Android Gradle Plugin 9.1.1. Core Camera features are run with `androidx.camera:*` v1.5.0.

## SECTION 10: IOS SETUP
The iOS application targets iOS `12.4` and relies on an underlying `Podfile` combining React Native configurations alongside WatermelonDB specifications.
**Permissions required in `Info.plist`:**
* `NSCameraUsageDescription`: "Auralock requires camera access for offline face recognition and liveness detection."
* `NSLocationWhenInUseUsageDescription`: "Auralock requires location access to securely log the geolocation of biometric verifications."
No external packages directly handle face detection models within SwiftUI currently, relying directly on `Vision` and `CoreImage` libraries natively implemented inside Xcode structures (`Auralock.xcodeproj`).

## SECTION 11: MODEL FILES
Model files are found in `src/assets/models/`.
* `scrfd_500m_int8.tflite` - Detects the fundamental boundaries of faces relative to the frame.
* `mobilefacenet_arcface_int8.tflite` - Computes facial vectors extracting 128-dimensional embedding parameters for offline MobileFaceNet cosine distance thresholds.
* `silent_face_int8.tflite` - Supports Silent liveness (Passive mode anti-spoofing).

## SECTION 12: INSTRUCTION MESSAGES
Messages pushed directly to users based on pipeline states:
* "Position your face inside the bounding box and blink naturally." - (Initial interactive verification request trigger)
* "Position your face in the oval" - (No faces detected)
* "Multiple people detected. Only one person at a time" - (Face instances > 1)
* "Move closer to the camera" - (`TOO_SMALL` trigger condition)
* "Center your face in the oval" - (`NOT_CENTERED` trigger condition)
* "Keep your head straight" - (`TILTED` trigger condition)
* "Too dark. Move to a well-lit area" - (`TOO_DARK` trigger condition)
* "Too bright. Avoid direct sunlight or glare" - (`HARSH_SUNLIGHT` trigger condition)
* "Please blink naturally" - (Triggering `BLINK` challenge)
* "Please smile naturally" - (Triggering `SMILE` challenge)
* "Slowly turn your head" - (Triggering `TURN_HEAD` challenge)
* "Identity verified" / "Identity Verified!" - (Post-challenge compiling transition)
* "Looking for your face..." - (iOS Initial)
* "Great! Next: Smile" / "Great! Next: Blink" - (iOS Challenge Transitions)
* "Access Granted" / "Access Denied" - (iOS completion states)

## SECTION 13: LIVENESS CAMERA ICON STATES
Managed via `LivenessCameraIconView.kt` (`State` class) running custom path stroke animations mapping to:
* `IDLE`: Solid inner ring and static lens (`Color.GRAY`).
* `PASSIVE_CHECK`: A spinning outer ring simulating active calculation (`#FFC107` Amber).
* `BLINK`: Eyelid closing animation mapping quadratic bezier path bounds (`#4CAF50` Green).
* `SMILE`: Curves up a 180° rounded curve (`#4CAF50` Green).
* `TURN_LEFT`: Horizontal path sweep forming a left-pointing arrow configuration (`#4CAF50` Green).
* `TURN_RIGHT`: Horizontal path sweep forming a right-pointing arrow configuration (`#4CAF50` Green).
* `COMPLETE`: Generates a scaled checkmark mark animation bridging paths (`#4CAF50` Green).
* `FAILED`: Generates an X mark and translates paths via lateral shake (`#F44336` Red).

## SECTION 14: TESTING
The codebase includes unit test files established via Robolectric and Roborazzi (indicated by `gradle/libs.versions.toml`).
Specific files under `app/src/test/java/com/example/`:
* `ExampleRobolectricTest.kt`
* `ExampleUnitTest.kt`
* `GreetingScreenshotTest.kt`
These files natively address JVM and Roborazzi testing frameworks ensuring UI snapshots and repository models function correctly without an active emulator.

## SECTION 15: KNOWN LIMITATIONS AND ASSUMPTIONS
* The telemetry limits (EAR, MAR, Roll) are established manually via constants (`earThreshold = 0.24`, `marThreshold = 0.40`).
* Tilt tests utilize rudimentary sine approximations calculated between dual landmarks, without a true 3D pose extraction mechanism.
* Models act directly upon synthetic deterministic metrics rather than feeding directly into real Tensor Buffers per `FaceRecognitionEngine.swift` mock definitions.
* The iOS logic calculates similarity scores mapped to a threshold of `0.65`, whereas Android compares distances strictly above `0.75f`.

## SECTION 16: LICENSE AND CONTRIBUTING
The repository does not contain an established `LICENSE` or `CONTRIBUTING` file mapping open-source declarations.
# Auralock
