# SASSA Shield

SASSA Shield is an Android application designed to help protect South African users from SASSA-related and government-impersonation scams. The app monitors incoming SMS messages, classifies them using an AI-first approach with an offline heuristic fallback, alerts the user when suspicious content is detected, and stores detected scam events locally. It also includes call-screening support to warn about or block suspicious callers using local rules, cached results, and an optional phone-intelligence API.

> **Project type:** Android (Kotlin, Jetpack Compose)  
> **Package:** `com.example.myapplication`  
> **Minimum SDK:** 24 (Android 7.0)  
> **Target / Compile SDK:** 36

## 1. Project Overview

Social grant scams often impersonate SASSA, the South African government, or related services. Common tactics include fake prize draws, fake relief-grant approvals, urgent requests to click unofficial links, and requests for sensitive information such as ID numbers, card numbers, PINs, OTPs, or banking details.

SASSA Shield addresses this problem by:

- Receiving incoming SMS messages in real time.
- Analyzing message content for scam indicators.
- Using **LLM-based classification first** (Firebase AI / Gemini) when available.
- Falling back to **offline heuristic detection** when the AI service is unavailable, times out, or returns an unusable response.
- Displaying high-priority scam notifications.
- Saving detected scam messages to a local Room database for review.
- Screening incoming calls and applying block/warn decisions based on manual blocklists, cached risk data, international-number heuristics, and optional third-party phone validation.

## 2. Key Features

### SMS Scam Detection
- Listens for incoming SMS using `BroadcastReceiver` (`SMS_RECEIVED`).
- **LLM-always with offline fallback** flow:
  1. Attempts AI classification via `ScamAiClassifier`.
  2. If AI classification fails, times out (12 seconds), or cannot be parsed, uses `ScamDetector` heuristic analysis.
- Generates a user-facing reason and labels the detection source in saved/alert text:
  - `(AI)` for LLM classification.
  - `(offline check)` for heuristic fallback.
- Shows a notification for detected scams.
- Stores detected messages in the local `scam_messages` table.

### Offline Heuristic Detection
The heuristic detector scores messages using patterns observed in real scam samples, including:

- SASSA/government/grant keywords.
- Requests for sensitive data (ID number, card details, PIN, OTP, password, banking details, SIM details).
- Prize, draw, relief-grant, or cash-withdrawal bait.
- Urgent/action language (e.g., urgently, reply, click/apply/approve).
- Suspicious or shortened links.
- Non-official links in SASSA/government-branded contexts. Official guidance reflected in detection logic: legitimate SASSA/government links are expected to end in `*.gov.za` (e.g., `sassa.gov.za`, `dsd.gov.za`).

### Call Screening
- Uses Android `CallScreeningService` to screen incoming calls.
- Decision flow:
  1. **Manual blocklist** (offline): blocks numbers manually blocked by the user.
  2. **Local cache** (offline): blocks/warns based on previously stored high-risk scam calls.
  3. **International heuristic** (offline): flags non-`+27` international callers as potential risk in the SASSA context.
  4. **Abstract Phone Intelligence API** (network, optional): validates phone numbers and uses risk assessment/abuse signals when an API key is configured.
- Can block calls, warn the user, and record call risk data locally.

### User Interface (Jetpack Compose)
- **Dashboard**
  - Protection status for SMS permissions.
  - Manual link/message check using the heuristic detector.
  - Call protection status and role request for call screening.
  - Permission prompts for SMS and notifications.
  - Common scam phrase guidance.
- **History**
  - Scam Messages tab: lists detected SMS scams and supports deletion.
  - Calls tab: lists suspicious/blocked calls and supports manual block/unblock actions.

### Local Data Storage
Room database (`scam_database`, version 3) stores:

- `scam_messages`: detected scam SMS records.
- `scam_calls`: screened call records and risk metadata.
- `manual_blocks`: user-managed blocked phone numbers.

## 3. System Architecture

```text
Incoming SMS
   │
   ▼
SmsReceiver (BroadcastReceiver)
   │
   ├─► ScamAiClassifier (Firebase AI / Gemini, few-shot prompt)
   │        │
   │        ├─ success ─► DetectionResult(source="llm")
   │        └─ failure/timeout/unparseable ─► fallback
   │
   └─► ScamDetector (offline heuristics)
            │
            ▼
      DetectionResult(source="heuristic-fallback")
            │
            ├─ if scam ─► NotificationHelper (high-priority alert)
            └─ if scam ─► Room DB (ScamMessage)

Incoming Call
   │
   ▼
MyCallScreeningService
   │
   ├─ Manual blocklist (Room)
   ├─ Cached scam call (Room)
   ├─ International heuristic (+27 context)
   └─ Optional Abstract Phone Intelligence API
            │
            ▼
   Block / Warn / Allow + Notification + Cache
```

## 4. Technology Stack

| Area | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose, Material 3, Navigation Compose |
| Architecture components | AndroidX Lifecycle, ViewModel Compose (dependency present) |
| Database | Room (`runtime`, `ktx`, KSP compiler) |
| Asynchronous work | Kotlin Coroutines |
| AI classification | Firebase AI (`com.google.firebase:firebase-ai`) |
| Networking | Retrofit, Gson converter, OkHttp logging interceptor |
| Phone intelligence | Abstract Phone Intelligence API (optional) |
| Build system | Gradle Kotlin DSL, Android Gradle Plugin, KSP |
| Services/receivers | `BroadcastReceiver`, `CallScreeningService` |

## 5. Project Structure

```text
sassaShield/
├── app/
│   ├── build.gradle.kts
│   ├── google-services.json                 # Firebase config (project-specific)
│   ├── proguard-rules.pro
│   └── src/
│       └── main/
│           ├── AndroidManifest.xml
│           └── java/com/example/myapplication/
│               ├── MainActivity.kt          # Compose UI: dashboard, history, permissions
│               ├── SmsReceiver.kt           # SMS entry point; LLM-first + fallback
│               ├── ScamAiClassifier.kt      # Firebase AI/Gemini few-shot classifier
│               ├── ScamDetector.kt          # Offline heuristic scam scoring
│               ├── MyCallScreeningService.kt# Call screening/block/warn logic
│               ├── NotificationHelper.kt    # Scam/call alert notifications
│               ├── data/
│               │   ├── ScamDatabase.kt      # Room database
│               │   ├── ScamDao.kt           # DAO for messages/calls/manual blocks
│               │   ├── ScamMessage.kt       # SMS scam entity
│               │   ├── ScamCall.kt          # Call risk entity
│               │   ├── ManualBlock.kt       # Manual block entity
│               │   └── ScamRepository.kt    # Legacy SharedPreferences repo (present)
│               ├── repository/
│               │   └── PhoneRepository.kt   # Abstract API integration
│               ├── network/
│               │   ├── AbstractPhoneService.kt
│               │   └── AbstractPhoneResponse.kt
│               └── ui/theme/                # Compose theme files
├── build.gradle.kts
├── settings.gradle.kts
├── gradle/
│   └── libs.versions.toml                   # Version catalog
└── local.properties                         # Local SDK path / API keys (not committed)
```

## 6. Core Components

| Component | Responsibility |
|---|---|
| `SmsReceiver` | Receives `SMS_RECEIVED` broadcasts, invokes AI classifier first, falls back to heuristics, notifies user, persists scam messages. |
| `ScamAiClassifier` | Builds a few-shot prompt from known scam patterns, calls Firebase AI Gemini model, parses JSON output into `DetectionResult`. |
| `ScamDetector` | Performs deterministic heuristic scoring over message text and URLs; used as offline fallback and by dashboard link check. |
| `MyCallScreeningService` | Screens calls using manual blocklist, cached risk, international heuristic, and optional Abstract API phone validation. |
| `NotificationHelper` | Creates high-priority notification channel/alerts for scam SMS and call events. |
| `ScamDatabase` / `ScamDao` | Local persistence for scam messages, scam calls, and manual blocks. |
| `MainActivity` | Main UI, permission handling, call-screening role request, link checker, scam/call history views. |
| `PhoneRepository` | Calls Abstract Phone Intelligence API when `ABSTRACT_API_KEY` is configured. |

## 7. AI Detection Design

### Method Used
For the current prototype/data scale, the project uses **few-shot LLM classification** rather than training a custom model from scratch. This is appropriate when only a small labeled set is available (e.g., ~15 scam examples), because training/fine-tuning a reliable classifier usually requires a much larger, balanced dataset.

### Runtime Behavior
- **Primary:** Firebase AI / Gemini model (`gemini-2.0-flash`, JSON response expected).
- **Fallback:** heuristic `ScamDetector` result.
- **Timeout:** AI classification uses a 12-second timeout to avoid blocking SMS handling indefinitely.
- **Failure handling:** exceptions, missing text, or JSON parsing failures trigger offline fallback.

### Example Scam Patterns Used for Few-Shot Guidance
The AI prompt includes patterns derived from collected samples, such as:

- Fake SASSA career/application links.
- Prize/draw scams requesting ID/card/PIN details.
- Fake relief-grant links (including unofficial/shortened URLs).
- Impersonation accounts requesting personal/SIM/approval details.
- Fake government relief/withdrawal prompts.
- Impersonation of authorities requesting personal information.

## 8. Permissions and Android Requirements

Declared in `AndroidManifest.xml`:

| Permission / Role | Purpose |
|---|---|
| `RECEIVE_SMS` | Receive incoming SMS messages for scam analysis. |
| `POST_NOTIFICATIONS` | Display scam/call alerts (runtime permission on Android 13+). |
| `READ_PHONE_STATE` | Supports telephony-related functionality/call context. |
| `INTERNET` | Required for Firebase AI classification and optional Abstract phone API calls. |
| `BIND_SCREENING_SERVICE` / `CallScreeningService` | Enables call screening. App may request the call-screening role. |
| `BROADCAST_SMS` (receiver permission) | Used for SMS broadcast receiver protection context. |

### Important Distribution Note
Apps that monitor SMS/calls are subject to strict platform and store policies (especially Google Play). Depending on distribution goals, the app may need to qualify under allowed use cases and provide clear user disclosures, privacy policy details, and permission justification.

## 9. Setup and Configuration

### Prerequisites
- Android Studio (recent stable version recommended).
- Android SDK for API 36 / build tools compatible with the project.
- JDK compatible with the Android Gradle/Kotlin setup.
- A physical Android device is recommended for SMS/call behavior testing (emulators have limitations).

### Clone / Open
1. Open the `sassaShield` folder in Android Studio.
2. Let Gradle sync dependencies.

### Firebase AI Setup
The project includes `com.google.firebase:firebase-ai` and applies Google Services.

1. Create/configure a Firebase project and Android app with package `com.example.myapplication` (or update the app/package and config consistently).
2. Place `google-services.json` in `app/`.
3. Enable the Firebase AI / Gemini capability required by the SDK for your Firebase project.
4. Ensure network access is available on test devices for LLM classification.

If Firebase AI is unavailable, SMS classification should fall back to heuristics.

### Optional: Abstract Phone Intelligence API
Call screening can use Abstract API for phone validation/risk assessment.

Add to `local.properties`:

```properties
ABSTRACT_API_KEY=your_api_key_here
```

The key is exposed to the app as `BuildConfig.ABSTRACT_API_KEY`. If absent, `PhoneRepository` returns `null` and call screening continues with local/manual/cached/heuristic logic.

### Local Android SDK
`local.properties` must contain a valid SDK path, e.g.:

```properties
sdk.dir=C\:\\Users\\<user>\\AppData\\Local\\Android\\Sdk
```

## 10. Build and Run

### Android Studio
- Select a device/emulator.
- Run the `app` configuration.

### Gradle (Windows)
```powershell
.\gradlew.bat :app:assembleDebug
```

Compile Kotlin only:
```powershell
.\gradlew.bat :app:compileDebugKotlin
```

Release build tasks (signing configuration required for distribution):
```powershell
.\gradlew.bat :app:assembleRelease
.\gradlew.bat :app:bundleRelease
```

> Build note: ensure `JAVA_HOME` is set and a compatible JDK is available in the environment.

## 11. Testing and Verification

### Manual Verification
1. Install and launch the app.
2. Grant SMS and notification permissions.
3. Request/enable call screening role if testing call protection.
4. Send test SMS messages containing scam indicators (e.g., SASSA impersonation + unofficial link, requests for PIN/OTP/ID/card details, fake grant/prize text).
5. Confirm:
   - notification is shown for detected scams;
   - scam appears in History → Messages;
   - offline fallback still works when network/AI is unavailable.
6. Use the Dashboard link checker with official and unofficial links.
7. Test call screening scenarios (manual block, cached risk, international number, API-configured number validation).

### Automated Tests
Current test scaffolding includes:
- `app/src/test/java/com/example/myapplication/ExampleUnitTest.kt`
- `app/src/androidTest/java/com/example/myapplication/ExampleInstrumentedTest.kt`

Recommended additions:
- Unit tests for `ScamDetector` (keywords, URL rules, shortened links, `.gov.za` validation, sensitive-data requests).
- Unit tests for AI JSON parsing/fallback behavior.
- Instrumented tests for Room DAO operations.
- Receiver/service behavior tests with Android test APIs/mocks.

## 12. Privacy, Security, and Ethical Considerations

- **Sensitive data:** Scam messages may contain personal information. Handle stored data carefully and minimize exposure.
- **AI data transmission:** LLM classification sends message content/sender context to the configured AI backend. For a production/privacy-sensitive deployment, consider user consent, privacy disclosures, redaction, regional compliance, or an on-device model.
- **Secrets:** Do not commit real API keys. Keep `ABSTRACT_API_KEY` in `local.properties`; review Firebase config handling for your distribution model.
- **False positives/negatives:** No classifier is perfect. The app should present results as risk alerts, not absolute proof.
- **Permissions transparency:** Clearly explain why SMS/call/notification permissions are required.

## 13. Limitations

- Few-shot LLM performance depends on prompt quality, model behavior, and network availability.
- Heuristic detection can miss novel scam phrasing and may flag borderline messages.
- AI responses require network access and Firebase AI availability.
- Call screening risk depends on local data quality and optional third-party API coverage.
- The dashboard link checker currently uses `ScamDetector` directly (heuristic path), while incoming SMS uses LLM-first with fallback.
- Current automated test coverage is minimal and should be expanded for a production report/submission.

## 14. Future Improvements

- Collect and label a balanced dataset (scam and legitimate messages) with consent and PII redaction.
- Train/export an on-device text classifier (e.g., TensorFlow Lite) for privacy and offline performance.
- Add confidence thresholds and a hybrid policy (heuristics + LLM only for uncertain cases) if data/privacy/cost require it.
- Improve sender reputation checks and number normalization.
- Add multilingual support (South African languages/mixed-language scams).
- Add secure export/reporting features for case documentation.
- Expand unit/instrumentation/UI test coverage.
- Add privacy policy, consent flow, and production-ready permission onboarding.

## 15. Academic / Report Notes

This project demonstrates a practical mobile security solution combining:

- **Rule-based detection** for explainable, offline-capable screening.
- **LLM few-shot classification** for flexible understanding of scam language with limited labeled data.
- **Defense-in-depth/fallback design** where AI is preferred but heuristics maintain availability.
- **Android platform integration** through SMS receivers, notifications, local persistence, and call screening.

The current implementation is suitable as a prototype/MVP and a foundation for further research into scam detection, mobile privacy, and AI-assisted user protection.
