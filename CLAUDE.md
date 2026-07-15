# BugReel Pro — Claude Code Execution Guide

> **READ THIS ENTIRE FILE BEFORE WRITING ANY CODE.**
> This is the single source of truth for every architectural decision, naming rule, sprint scope,
> and implementation constraint in this project. If anything here conflicts with a suggestion from
> prior context or training, this file wins. Update this file before starting any sprint that
> changes the structure, patterns, or versions below.

---

## 0. Session Protocol

### How every Claude Code session starts
1. Read this file top to bottom — do not skip sections.
2. Confirm the current sprint number from the human.
3. Check the tech stack versions in Section 2 against official sources if any sprint has passed since last verification.
4. Build only the files listed for the active sprint — nothing from future sprints.
5. Run `./gradlew test` and `./gradlew lint` before declaring a sprint done.
6. Report stop condition status explicitly. Do not start the next sprint until the human confirms checklist sign-off.

### How every Claude Code session ends
- State which stop conditions passed and which did not.
- If any stop condition failed: list the failing items clearly. Do not suppress failures.
- Never start Sprint N+1 in the same session as Sprint N without explicit human approval.
- Commit with the convention in Section 21 before ending the session.

### One sprint per session rule
Never build ahead into future sprints even if the current sprint finishes early.
The sprint checklist in PRD V4.0 requires a human device-test sign-off that Claude Code cannot perform.
Finishing early means writing better tests, not starting the next sprint.

---

## 1. Project Context

**Product:** BugReel Pro — a native Android app for professional crowdtesters on platforms such
as uTest, Testlio, and Test.io. The app captures screen recordings and device events, builds
structured bug reports, scores them with a deterministic quality engine, and generates AI-assisted
copy (titles, summaries, repro steps).

**The single most important business rule:**
`OutcomeFeedbackEntity` must exist in the Room schema at version 1, Sprint 1, Day 1.
The entire long-term defensibility of the company depends on the accepted-vs-rejected outcome
training loop. It is not a V2 feature. It is the company. If it is absent from the schema, fix it
before touching any other file.

**Package root:** `com.bugreel.pro`

**Companion documents** (human has these — do not recreate them):
- `ARCHITECTURE.md` — Architectural decisions with rationale (ADR-001 through ADR-005)
- `BugReel_Pro_Final_PRD_V4.docx` — Sprint checklists, success metrics, risk register
- `BugReel_Pro_Sprint0_Validation_Spike_Runbook.docx` — Sprint 0 gate memos

---

## 2. Tech Stack — Verify These Before Every Sprint

Run the verification commands before Sprint 1 and again if more than two weeks have passed
since the last session. Never assume versions from a previous session are still current.

| Layer | Technology | Version | Verify at |
|---|---|---|---|
| Language | Kotlin | 2.x latest stable | kotlinlang.org/docs/releases.html |
| UI | Jetpack Compose | BOM 2026.x latest | developer.android.com/jetpack/compose/bom |
| Navigation | Compose Navigation | 2.8.x | developer.android.com/jetpack/androidx/releases/navigation |
| DI | Hilt | 2.52+ | dagger.dev/hilt/gradle-setup |
| DI Processor | KSP | latest for Kotlin 2.x | github.com/google/ksp/releases |
| Local DB | Room | 2.7.x | developer.android.com/jetpack/androidx/releases/room |
| Preferences | DataStore (Proto) | 1.1.x | developer.android.com/topic/libraries/architecture/datastore |
| Background | WorkManager | 2.10.x | developer.android.com/jetpack/androidx/releases/workmanager |
| Sync | Firebase Firestore | 25.x | firebase.google.com/support/release-notes/android |
| Auth | Firebase Auth | 23.x | (same) |
| Storage | Firebase Cloud Storage | 21.x | (same) |
| Crash | Firebase Crashlytics | 19.x | (same) |
| AI (text) | OpenAI gpt-4.1-mini / gpt-4.1 | via OkHttp — no SDK | platform.openai.com/docs |
| AI (vision) | OpenAI vision endpoint | same OkHttp client | (same) |
| Images | Coil | 3.x | coil-kt.github.io/coil |
| Serialization | kotlinx.serialization | 1.7.x | github.com/Kotlin/kotlinx.serialization/releases |
| Unit Testing | JUnit5 + MockK + Turbine | latest | mannodermaus.github.io/android-junit5 |
| Device Testing | Espresso + UIAutomator | latest | developer.android.com |
| Build | Gradle KTS | 8.5+ | gradle.org/releases |
| IDE | Android Studio Quail 1 | 2026.1.1 | developer.android.com/studio |
| JDK | Temurin JDK 17 | 17.0.19+ | adoptium.net/temurin/releases/?version=17 |
| minSdk | 28 (Android 9) | — | — |
| targetSdk / compileSdk | 35 | — | — |

**BuildConfig constants — set in `app/build.gradle.kts` `defaultConfig` block:**
```
AI_MODEL_STANDARD     = "gpt-4.1-mini"
AI_MODEL_SAFETY       = "gpt-4.1"
QUALITY_SCORE_UPGRADE = 70
AI_RATE_LIMIT_RPM     = 20
SYNC_INTERVAL_MINUTES = 30
FLYWHEEL_ENABLED      = true
```

---

## 3. Directory Structure

Every directory below must exist before Sprint 1 begins (scaffold them with `mkdir -p`).
Claude Code creates files inside this structure — never invents new top-level directories.

```
bugreel-pro/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/bugreel/pro/
│   │   │   │   ├── BugReelApplication.kt
│   │   │   │   ├── MainActivity.kt
│   │   │   │   ├── navigation/
│   │   │   │   │   ├── AppNavGraph.kt
│   │   │   │   │   └── Screen.kt
│   │   │   │   ├── core/
│   │   │   │   │   ├── di/
│   │   │   │   │   │   ├── AppModule.kt
│   │   │   │   │   │   ├── DatabaseModule.kt
│   │   │   │   │   │   ├── NetworkModule.kt
│   │   │   │   │   │   └── FirebaseModule.kt
│   │   │   │   │   ├── data/
│   │   │   │   │   │   ├── local/
│   │   │   │   │   │   │   ├── BugReelDatabase.kt
│   │   │   │   │   │   │   ├── Converters.kt
│   │   │   │   │   │   │   ├── dao/
│   │   │   │   │   │   │   │   ├── ReportDao.kt
│   │   │   │   │   │   │   │   ├── EvidenceDao.kt
│   │   │   │   │   │   │   │   ├── DraftDao.kt
│   │   │   │   │   │   │   │   ├── QualityScoreDao.kt
│   │   │   │   │   │   │   │   ├── AiOutputDao.kt
│   │   │   │   │   │   │   │   └── OutcomeFeedbackDao.kt   ← FLYWHEEL
│   │   │   │   │   │   │   └── entity/
│   │   │   │   │   │   │       ├── ReportEntity.kt
│   │   │   │   │   │   │       ├── EvidenceEntity.kt
│   │   │   │   │   │   │       ├── DraftEntity.kt
│   │   │   │   │   │   │       ├── QualityScoreEntity.kt
│   │   │   │   │   │   │       ├── AiOutputEntity.kt
│   │   │   │   │   │   │       ├── DeviceProfileEntity.kt
│   │   │   │   │   │   │       └── OutcomeFeedbackEntity.kt ← FLYWHEEL
│   │   │   │   │   │   ├── remote/
│   │   │   │   │   │   │   ├── firebase/
│   │   │   │   │   │   │   │   ├── FirestoreSource.kt
│   │   │   │   │   │   │   │   └── StorageSource.kt
│   │   │   │   │   │   │   └── ai/
│   │   │   │   │   │   │       ├── AIService.kt
│   │   │   │   │   │   │       ├── OpenAIService.kt
│   │   │   │   │   │   │       ├── PromptRepository.kt
│   │   │   │   │   │   │       ├── DefaultPromptRepository.kt
│   │   │   │   │   │   │       └── dto/
│   │   │   │   │   │   └── sync/
│   │   │   │   │   │       ├── SyncWorker.kt
│   │   │   │   │   │       └── SyncMapper.kt
│   │   │   │   │   ├── domain/
│   │   │   │   │   │   ├── model/
│   │   │   │   │   │   │   ├── BugReport.kt
│   │   │   │   │   │   │   ├── BugReportDraft.kt
│   │   │   │   │   │   │   ├── DeviceProfile.kt
│   │   │   │   │   │   │   ├── EvidenceBundle.kt
│   │   │   │   │   │   │   ├── QualityScore.kt
│   │   │   │   │   │   │   ├── AiOutput.kt
│   │   │   │   │   │   │   ├── OutcomeFeedback.kt          ← FLYWHEEL
│   │   │   │   │   │   │   ├── CoachingHint.kt
│   │   │   │   │   │   │   └── enums/
│   │   │   │   │   │   │       ├── Severity.kt
│   │   │   │   │   │   │       ├── Frequency.kt
│   │   │   │   │   │   │       ├── ReportStatus.kt
│   │   │   │   │   │   │       ├── PlatformOutcome.kt      ← FLYWHEEL
│   │   │   │   │   │   │       └── TestPlatform.kt
│   │   │   │   │   │   └── repository/
│   │   │   │   │   │       ├── ReportRepository.kt
│   │   │   │   │   │       ├── DraftRepository.kt
│   │   │   │   │   │       └── OutcomeFeedbackRepository.kt ← FLYWHEEL
│   │   │   │   │   └── util/
│   │   │   │   │       ├── OemCompat.kt
│   │   │   │   │       ├── PermissionManager.kt
│   │   │   │   │       └── Extensions.kt
│   │   │   │   └── feature/
│   │   │   │       ├── capture/
│   │   │   │       │   ├── data/, domain/, service/, ui/
│   │   │   │       ├── report/
│   │   │   │       │   ├── data/, domain/, ui/
│   │   │   │       ├── quality/
│   │   │   │       │   ├── data/, domain/, ui/
│   │   │   │       ├── ai/
│   │   │   │       │   ├── data/, domain/, ui/
│   │   │   │       └── analytics/
│   │   │   │           ├── data/, domain/, ui/
│   │   │   ├── res/
│   │   │   └── AndroidManifest.xml
│   │   ├── test/
│   │   └── androidTest/
│   └── build.gradle.kts
├── buildSrc/
│   └── Dependencies.kt
├── schemas/
├── CLAUDE.md               ← THIS FILE
├── ARCHITECTURE.md
├── build.gradle.kts
├── gradle.properties
├── gradlew / gradlew.bat
├── local.properties        ← GITIGNORED
└── .gitignore
```

---

## 4. Architecture — Four Inviolable Rules

Break these and the build becomes unmaintainable. They are checked via the Architecture Gates
in every sprint checklist.

### Rule 1 — Dependency direction
UI → Domain → Data. No reverse dependencies. No circular imports.
Domain has zero Android imports (`android.*`, `kotlinx.coroutines.*` excluded).
ViewModels call UseCases only. UseCases call Repository interfaces only.

### Rule 2 — Room is the source of truth
All UI state reads from Room via `Flow<Entity>`. Firebase writes happen only through `SyncWorker`.
No ViewModel or UseCase ever calls `FirestoreSource` or `StorageSource` directly.
Verify: `grep -rn "getFirestore\|Firebase" feature/ --include="*.kt"` must return 0 results.

### Rule 3 — OEM workarounds are centralised
Every `Build.MANUFACTURER`, `Build.BRAND`, MIUI version check, and OEM-specific workaround
lives exclusively in `OemCompat.kt`. Zero OEM checks anywhere else.
Verify: `grep -rn "Build.MANUFACTURER\|MIUI\|ONE UI" --include="*.kt" | grep -v OemCompat` must return 0.

### Rule 4 — No feature cross-imports
A feature package must not import another feature package. Shared types live in `core/domain/model/`.
Verify: `grep -rn "import com.bugreel.pro.feature\." feature/ --include="*.kt" | grep -v "self"` must return 0.

---

## 5. Naming Conventions

| Artifact | Suffix | Example |
|---|---|---|
| Screen composable | `Screen` | `ReportBuilderScreen.kt` |
| ViewModel | `ViewModel` | `ReportBuilderViewModel.kt` |
| UseCase | `UseCase` | `SubmitReportUseCase.kt` |
| Repository interface | `Repository` | `ReportRepository.kt` |
| Repository implementation | `RepositoryImpl` | `ReportRepositoryImpl.kt` |
| Room entity | `Entity` | `ReportEntity.kt` |
| Room DAO | `Dao` | `ReportDao.kt` |
| Domain model | No suffix | `BugReport.kt` |
| Hilt module | `Module` | `DatabaseModule.kt` |
| UI state sealed class | `UiState` | `ReportBuilderUiState` |
| UI event sealed class | `UiEvent` | `ReportBuilderUiEvent` |
| User intent sealed class | `Intent` | `ReportBuilderIntent` |
| Unit test | `Test` | `QualityScorerTest.kt` |
| Integration test | `IntegrationTest` | `ReportDaoIntegrationTest.kt` |

Test naming — backtick descriptions, plain English, behaviour-not-implementation:
```kotlin
// CORRECT
fun `empty report scores zero`()
fun `submit report stores to room and schedules firebase sync`()

// WRONG
fun testSubmitReport()
fun test_quality_score()
```

---

## 6. Core Domain Models

These are canonical. Room entities map TO these — they are not the entities themselves.
Domain models have zero Room, Firebase, or Android imports.

```kotlin
// core/domain/model/BugReport.kt
data class BugReport(
    val id: String,                      // UUID generated locally
    val title: String,
    val summary: String,
    val reproSteps: List<String>,
    val severity: Severity,
    val frequency: Frequency,
    val deviceProfile: DeviceProfile,
    val evidenceBundle: EvidenceBundle,
    val qualityScore: QualityScore?,
    val aiOutput: AiOutput?,
    val platform: TestPlatform,
    val status: ReportStatus,
    val createdAt: Instant,
    val updatedAt: Instant,
    val syncedAt: Instant?,
    val outcomeFeedback: OutcomeFeedback?    // THE FLYWHEEL — never null after Sprint 5
)

// core/domain/model/OutcomeFeedback.kt
// Every field in this model feeds the training pipeline. All are required.
data class OutcomeFeedback(
    val reportId: String,
    val platformOutcome: PlatformOutcome,      // ACCEPTED / REJECTED / PENDING
    val rejectionReason: String?,
    val reportedByTesterAt: Instant,
    val preAiQualityScore: Int,                // Score BEFORE AI assist was applied
    val postAiQualityScore: Int,               // Score AFTER AI assist was applied
    val testerOverrodeSuggestion: Boolean,     // Did tester change AI output before submitting?
    val aiSuggestionsAccepted: List<String>,   // Field names tester kept unchanged
    val aiSuggestionsRejected: List<String>    // Field names tester overwrote
)

// core/domain/model/QualityScore.kt
data class QualityScore(
    val total: Int,          // 0–100; deterministic weighted sum — NOT an AI call
    val reproduction: Int,   // max 25
    val evidence: Int,       // max 25
    val description: Int,    // max 20
    val severity: Int,       // max 15
    val metadata: Int,       // max 15
    val coaching: List<CoachingHint>
)

// core/domain/model/CoachingHint.kt
data class CoachingHint(
    val field: String,
    val message: String,     // Plain English. Max 120 chars. Sourced from CoachingHintStrings.kt
    val scoreImpact: Int     // Points this fix would add
)
```

---

## 7. Data Architecture — Room as Source of Truth

```
User action
  → UseCase → RepositoryImpl → Room write (immediate)
                              → UI updates via Flow<Entity> (immediate)
                              → WorkManager SyncWorker (async, background)
                                  → Firebase Firestore write (best-effort, with retry)
```

Firebase reads happen ONLY on first login (pull cloud history to Room) and explicit
user-triggered sync. Never during normal app operation.

### Room database registration — set in Sprint 1, never changed after

```kotlin
@Database(
    entities = [
        ReportEntity::class,
        EvidenceEntity::class,
        DraftEntity::class,
        QualityScoreEntity::class,
        AiOutputEntity::class,
        DeviceProfileEntity::class,
        OutcomeFeedbackEntity::class,   // ADR-002: Present from version=1. Non-negotiable.
    ],
    version = 1,
    exportSchema = true                 // Always true. Schema JSON committed to repo.
)
@TypeConverters(Converters::class)
abstract class BugReelDatabase : RoomDatabase() {
    abstract fun reportDao(): ReportDao
    abstract fun evidenceDao(): EvidenceDao
    abstract fun draftDao(): DraftDao
    abstract fun qualityScoreDao(): QualityScoreDao
    abstract fun aiOutputDao(): AiOutputDao
    abstract fun deviceProfileDao(): DeviceProfileDao
    abstract fun outcomeFeedbackDao(): OutcomeFeedbackDao
}
```

### SyncWorker constraints

```kotlin
// Constraints: NetworkType.CONNECTED
// Trigger: connectivity restored, app foreground after 30+ min, user pull-to-refresh
// Retry: exponential backoff, initial 30s, max 5 retries
// Conflict resolution: Room (local) wins. Last local write timestamp wins.
// On permanent failure: surface as non-blocking indicator in UI, not a crash
```

### Schema migration protocol

Every sprint that adds, removes, or renames an entity field must also add a `Migration` object.
Adding `OutcomeFeedbackEntity` in Sprint 1 at version=1 requires no migration.
Any change in Sprints 2–5 is a migration from version N to N+1.
Run `./gradlew :app:kaptDebugKotlin` after every entity change and commit the generated schema JSON.
An app update without a migration = data loss for all existing users = unacceptable.

---

## 8. Capture Engine Architecture

CaptureService is a ForegroundService. Nothing capture-related runs on the main thread.

```
CaptureService (ForegroundService; android:process=":capture")
├── MediaProjectionHandler    — screen recording via MediaProjectionManager
│   ├── Handles system consent dialog and OEM re-prompt variance
│   └── Runs in a dedicated coroutine scope tied to the Service lifecycle
├── BugReelEventLogger        — structured in-app event stream (no permissions needed)
│   ├── Captures: taps, swipes, screen transitions, timestamps from BugReel itself
│   └── This is the ALWAYS-AVAILABLE evidence layer — it works on every device
├── DeviceProfiler            — read-only device metadata (no special permissions)
├── BubbleOverlayManager      — floating capture trigger (ADR-005: dual-path from Day 1)
│   ├── PRIMARY path:  SYSTEM_ALERT_WINDOW floating bubble
│   └── FALLBACK path: PiP/notification-tile trigger (activate if Play policy returns RED)
└── LogCollector
    ├── Standard: BugReelEventLogger output (always available)
    └── Power-user opt-in: guided ADB / manual-paste flow
```

### OemCompat.kt — all OEM workarounds live here exclusively

```kotlin
object OemCompat {
    fun requiresManualBatteryOptimizationWhitelist(): Boolean
    fun overlayPermissionGrantIntent(context: Context): Intent
    fun foregroundServiceSurvivesBatterySaver(): Boolean
    fun mediaProjectionReprompts(): Boolean      // true on Samsung/MIUI
    fun needsPiPFallback(): Boolean
}
```

### BubbleOverlayManager mode injection

```kotlin
enum class BubbleMode { PRIMARY, FALLBACK }

class BubbleOverlayManager @Inject constructor(
    @Named("bubbleMode") private val mode: BubbleMode,
    ...
)
```
The mode is set once at app start based on `OemCompat.needsPiPFallback()` and the Play policy
outcome from Sprint 0 Track 2. It never switches at runtime.

---

## 9. AI Service Architecture

### Interfaces — inject these, never the implementations

```kotlin
// core/data/remote/ai/AIService.kt
interface AIService {
    suspend fun generateTitle(context: ReportContext): AIResult<String>
    suspend fun generateSummary(context: ReportContext): AIResult<String>
    suspend fun formatReproSteps(context: ReportContext): AIResult<List<String>>
    suspend fun validateSafety(context: ReportContext, output: AiOutput): SafetyResult
}

// core/data/remote/ai/PromptRepository.kt
// ALL prompt strings live here. Changing a prompt touches exactly one file.
interface PromptRepository {
    fun titlePrompt(context: ReportContext): String
    fun summaryPrompt(context: ReportContext): String
    fun reproStepsPrompt(context: ReportContext): String
    fun safetyValidationPrompt(context: ReportContext, output: AiOutput): String
}
```

### Model routing (ADR-003)

```kotlin
// NetworkModule.kt — never hardcode model strings elsewhere
BuildConfig.AI_MODEL_STANDARD  // gpt-4.1-mini — Title, Summary, ReproSteps
BuildConfig.AI_MODEL_SAFETY    // gpt-4.1 — SafetyValidator only (reliability > cost here)
```

### SafetyValidator failure criteria (ADR-004) — all four are testable

A safety check returns `SafetyResult.Blocked` if the AI output:
1. References a UI element not present in the captured screenshot evidence
2. Claims a severity not supported by the evidence bundle (e.g. Critical with no crash log)
3. Contains repro steps not derivable from the BugReelEventLogger gesture log
4. States device metadata inconsistent with DeviceProfile

On FAIL:
- Return the original unmodified tester input
- Display a clear message: "AI suggestion was blocked — showing your original text"
- Log the block to analytics (feeds prompt improvement)
- Never silently discard, auto-correct, or display flagged content

On PASS:
- Record `testerOverrodeSuggestion = false` in OutcomeFeedback initially
- If tester edits the AI output, set `testerOverrodeSuggestion = true`

### AI rate limiter

```kotlin
// Soft limit: BuildConfig.AI_RATE_LIMIT_RPM requests/minute per user
// On limit hit: return RateLimitExceeded with retryAfterSeconds
// Surface to UI: "You've reached the AI limit. Try again in X seconds."
// Log hits to analytics — signals whale-user risk to the unlimited tier
```

---

## 10. Quality Scorer Contract

```kotlin
// feature/quality/domain/QualityScorer.kt
//
// THIS IS A PURE FUNCTION. Enforce these constraints:
//   Zero IO operations
//   Zero coroutines / suspend functions
//   Zero database access
//   Zero AI calls
//   Zero Android imports (no Context, no Uri)
//   Same input ALWAYS produces same output (deterministic — test this with 1,000 iterations)
//
// Scoring weights:
//   reproduction  25  — steps present, numbered, ≥2 steps, clear language
//   evidence      25  — recording present (15), screenshots (7), logs (3)
//   description   20  — summary >50 chars (10), <500 chars (5), no placeholder text (5)
//   severity      15  — severity filled (10), consistent with evidence (5)
//   metadata      15  — device profile complete (10), platform template populated (5)
//
// Coaching hints:
//   Generated for every dimension scoring <80% of its max
//   Max 3 hints shown at once — highest scoreImpact first
//   All hint strings live in CoachingHintStrings.kt — never inline in QualityScorer
//
// Verify: grep -rn "AIService\|suspend\|Flow\|Context" feature/quality/domain/QualityScorer.kt
//         must return 0 results
```

---

## 11. State Management Pattern

Every ViewModel follows this pattern exactly. Do not invent alternatives.

```kotlin
@HiltViewModel
class ExampleViewModel @Inject constructor(
    private val exampleUseCase: ExampleUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow<ExampleUiState>(ExampleUiState.Loading)
    val uiState: StateFlow<ExampleUiState> = _uiState.asStateFlow()

    // One-shot events (navigation, Snackbar) — SharedFlow, not StateFlow
    private val _events = MutableSharedFlow<ExampleUiEvent>()
    val events: SharedFlow<ExampleUiEvent> = _events.asSharedFlow()

    fun onIntent(intent: ExampleIntent) {
        viewModelScope.launch { /* handle intent */ }
    }
}

// UiState  = what the screen looks like     (sealed class, collected with collectAsStateWithLifecycle)
// UiEvent  = fire-and-forget side effects   (sealed class, collected in LaunchedEffect)
// Intent   = user actions flowing in        (sealed class, sent from Screen to ViewModel)
```

---

## 12. Testing Architecture

### Unit tests (`src/test/`) — JUnit5 + MockK + Turbine

- File naming: `[ClassName]Test.kt`
- Backtick test names, plain English
- Never mock Room — use in-memory Room database for any test touching persistence
- Always mock: Firebase (`FirestoreSource`, `StorageSource`), `AIService`, external camera

```kotlin
class QualityScorerTest {
    @Test fun `empty report scores zero`() { ... }
    @Test fun `same input always produces same output`() {
        val report = buildTestReport()
        val results = (1..1000).map { scorer.score(report) }
        assert(results.all { it == results.first() })
    }
    @Test fun `coaching hints generated for sub-score below 80 percent max`() { ... }
}
```

### Integration tests (`src/androidTest/`) — real Room, mocked Firebase + AI

```kotlin
// Always use Room's in-memory builder for integration tests
val db = Room.inMemoryDatabaseBuilder(context, BugReelDatabase::class.java)
    .allowMainThreadQueries()
    .build()
```

### Coverage targets

| Layer | Minimum | Enforcement |
|---|---|---|
| `QualityScorer` | 100% | Every branch — it's a pure function, no excuse |
| UseCases | 90% | JUnit5 unit tests |
| Repository implementations | 85% | Integration tests with real in-memory Room |
| ViewModels | 80% | JUnit5 + Turbine |
| Compose screens | 60% | Compose testing library |
| `CaptureService` | Manual device tests | Sprint 1 checklist |

---

## 13. Sprint 0 — Validation Spike (COMPLETE — Reference Only)

Sprint 0 is done. Four gate memos were filed. Key decisions that flow into all subsequent sprints:

- **Track 1 (Capture):** Log Collector defaults to `BugReelEventLogger` (in-app events).
  Full logcat access not available on non-rooted devices. ADB path is power-user opt-in only.
- **Track 2 (Play Policy):** Accessibility Service justification submitted to Closed Testing.
  `BubbleOverlayManager` MUST have both PRIMARY and FALLBACK paths from Sprint 1 (ADR-005).
- **Track 3 (Legal):** Tester consent flow required before Private Alpha. AI training opt-in
  is a separate checkbox — never bundled with service consent.
- **Track 4 (Economics):** Text-only AI pipeline cost confirmed positive margin at P90.
  Vision/screenshot calls must be modelled separately. `AI_RATE_LIMIT_RPM = 20` is the
  soft cap that protects unlimited-tier margin.

Do not revisit Sprint 0 decisions mid-build unless a gate explicitly turns RED.

---

## 14. Sprint 1 — Capture Engine & Database Schema

**Scope:** Foundation. Everything that never changes after this sprint.
All 5 subsequent sprints depend on the Room schema created here.

### Session start prompt
```
Read CLAUDE.md. We are in Sprint 1: Capture Engine & Database Schema.
Build the files listed in Section 14 of CLAUDE.md in the order given.
Stop when all stop conditions in Section 14 are met. Do not start Sprint 2.
```

### Files to create — in this order

**Domain models first (no dependencies, pure Kotlin):**
1. `core/domain/model/enums/Severity.kt` — enum: CRITICAL, HIGH, MEDIUM, LOW, COSMETIC
2. `core/domain/model/enums/Frequency.kt` — enum: ALWAYS, SOMETIMES, RARELY, ONCE
3. `core/domain/model/enums/ReportStatus.kt` — enum: DRAFT, SUBMITTED, ACCEPTED, REJECTED
4. `core/domain/model/enums/PlatformOutcome.kt` — enum: ACCEPTED, REJECTED, PENDING ★ flywheel
5. `core/domain/model/enums/TestPlatform.kt` — enum: UTEST, TESTLIO, TEST_IO, OTHER
6. `core/domain/model/DeviceProfile.kt` — data class: brand, model, osVersion, ram, screenRes, networkType
7. `core/domain/model/EvidenceBundle.kt` — data class: recordingUri, screenshots, logText
8. `core/domain/model/QualityScore.kt` — data class per Section 6
9. `core/domain/model/CoachingHint.kt` — data class per Section 6
10. `core/domain/model/AiOutput.kt` — data class: title, summary, reproSteps, safetyPassed
11. `core/domain/model/OutcomeFeedback.kt` — data class per Section 6 ★ flywheel
12. `core/domain/model/BugReportDraft.kt` — data class: in-progress, unsaved report
13. `core/domain/model/BugReport.kt` — data class per Section 6

**Repository interfaces (domain layer, interfaces only):**
14. `core/domain/repository/ReportRepository.kt`
15. `core/domain/repository/DraftRepository.kt`
16. `core/domain/repository/OutcomeFeedbackRepository.kt` ★ flywheel

**Room entities:**
17. `core/data/local/entity/ReportEntity.kt`
18. `core/data/local/entity/EvidenceEntity.kt`
19. `core/data/local/entity/DraftEntity.kt`
20. `core/data/local/entity/QualityScoreEntity.kt`
21. `core/data/local/entity/AiOutputEntity.kt`
22. `core/data/local/entity/DeviceProfileEntity.kt`
23. `core/data/local/entity/OutcomeFeedbackEntity.kt` ★ flywheel

**Room DAOs:**
24. `core/data/local/dao/ReportDao.kt` — CRUD + `Flow<List<ReportEntity>>`
25. `core/data/local/dao/EvidenceDao.kt`
26. `core/data/local/dao/DraftDao.kt` — insert, update, getById, deleteById
27. `core/data/local/dao/QualityScoreDao.kt`
28. `core/data/local/dao/AiOutputDao.kt`
29. `core/data/local/dao/OutcomeFeedbackDao.kt` — insert(), queryByReportId() ★ flywheel

**Database:**
30. `core/data/local/Converters.kt` — TypeConverters: Instant↔Long, List<String>↔String, all enums
31. `core/data/local/BugReelDatabase.kt` — version=1, exportSchema=true, ALL 7 entities registered

**Utilities:**
32. `core/util/OemCompat.kt` — all methods stubbed with `TODO("Sprint 1 physical device testing")`
33. `core/util/PermissionManager.kt` — skeleton; all permission logic centralised here
34. `core/util/Extensions.kt` — common Kotlin extension functions

**Capture feature:**
35. `feature/capture/domain/CaptureRepository.kt` — interface
36. `feature/capture/data/DeviceProfiler.kt` — reads device metadata, returns DeviceProfile
37. `feature/capture/service/BugReelEventLogger.kt` — structured in-app event stream
38. `feature/capture/service/MediaProjectionHandler.kt` — screen recording logic
39. `feature/capture/service/BubbleOverlayManager.kt` — PRIMARY + FALLBACK modes (ADR-005)
40. `feature/capture/service/CaptureService.kt` — ForegroundService, `:capture` process
41. `feature/capture/data/CaptureRepositoryImpl.kt`
42. `feature/capture/domain/StartCaptureUseCase.kt`
43. `feature/capture/domain/StopCaptureUseCase.kt`
44. `feature/capture/ui/CaptureViewModel.kt`
45. `feature/capture/ui/CaptureScreen.kt`

**DI:**
46. `core/di/DatabaseModule.kt` — provides BugReelDatabase and all DAOs as singletons
47. `core/di/AppModule.kt` — provides OemCompat, PermissionManager

**App entry:**
48. `BugReelApplication.kt` — @HiltAndroidApp
49. `MainActivity.kt` — @AndroidEntryPoint, single-activity host
50. `navigation/Screen.kt` — sealed class of route strings
51. `navigation/AppNavGraph.kt` — NavHost with Capture as start destination

**Tests:**
52. `test/.../BugReelEventLoggerTest.kt` — all event types logged with correct timestamps
53. `test/.../OemCompatTest.kt` — all stub methods callable without crash or NPE
54. `test/.../DeviceProfilerTest.kt` — DeviceProfile non-null, no empty fields
55. `androidTest/.../DraftDaoIntegrationTest.kt` — insert + read round-trip (in-memory Room)
56. `androidTest/.../OutcomeFeedbackDaoIntegrationTest.kt` — insert + queryByReportId ★

### Key implementation notes
- `CaptureService` must declare `android:process=":capture"` in AndroidManifest.xml
- `BubbleOverlayManager` constructor takes a `BubbleMode` parameter — never hardcode PRIMARY
- `BugReelEventLogger` captures BugReel's own interaction events, NOT the test app's logcat
- `OutcomeFeedbackEntity` must have `preAiQualityScore` and `postAiQualityScore` fields even though they are 0 until Sprints 3 and 4 populate them respectively
- Hilt injection into `CaptureService` requires `@AndroidEntryPoint` on the Service class
- `Converters.kt` must handle nullable `Instant?` (for `syncedAt`)

### Stop conditions — Sprint 1
- `./gradlew test` passes — 0 failures
- `./gradlew lint` — 0 errors
- `OutcomeFeedbackEntity` confirmed in `@Database` entities list
- `BugReelDatabase.version == 1` confirmed
- Schema JSON exported to `schemas/` and present in working directory
- `BubbleOverlayManager` has both PRIMARY and FALLBACK implemented
- `grep -rn "getFirestore" feature/capture --include="*.kt"` returns 0 results

---

## 15. Sprint 2 — Report Builder & Firebase Sync

**Scope:** Report construction, evidence management, platform export, and Firebase sync.

### Session start prompt
```
Read CLAUDE.md. We are in Sprint 2: Report Builder & Firebase Sync.
Sprint 1 checklist has been signed off by the human. Build the files in Section 15.
Stop when all stop conditions are met. Do not start Sprint 3.
```

### Files to create — in this order

**Firebase interfaces (create the interfaces NOW; OpenAIService implementation is Sprint 4):**
1. `core/data/remote/ai/AIService.kt` — interface per Section 9 (implementation is Sprint 4)
2. `core/data/remote/ai/PromptRepository.kt` — interface per Section 9 (implementation is Sprint 4)

**Firebase data sources:**
3. `core/data/remote/firebase/FirestoreSource.kt` — sole class that writes to Firestore
4. `core/data/remote/firebase/StorageSource.kt` — sole class that writes to Firebase Storage

**Sync:**
5. `core/data/sync/SyncMapper.kt` — Entity ↔ Firestore document mapping
6. `core/data/sync/SyncWorker.kt` — WorkManager worker; the ONLY Firestore write path

**Domain models (new in Sprint 2):**
7. `core/domain/model/EvidenceBundle.kt` — (already exists from Sprint 1; verify and expand if needed)

**Report feature:**
8. `feature/report/domain/PlatformTemplateRepository.kt` — interface; template per TestPlatform
9. `feature/report/data/EvidenceManager.kt` — attaches MP4, PNG, TXT files; validates types
10. `feature/report/data/ZipGenerator.kt` — generates ZIP with defined folder structure
11. `feature/report/data/ReportRepositoryImpl.kt`
12. `feature/report/domain/BuildReportUseCase.kt`
13. `feature/report/domain/ExportReportUseCase.kt` — generates ZIP via ZipGenerator
14. `feature/report/ui/ReportBuilderViewModel.kt`
15. `feature/report/ui/ReportBuilderScreen.kt` — guided form, validation before submission
16. `feature/report/ui/ReportListScreen.kt` — list of reports; empty state REQUIRED
17. `feature/report/ui/EvidencePickerScreen.kt`

**DI:**
18. `core/di/NetworkModule.kt` — OkHttp client, AI base URL, BuildConfig model constants
19. `core/di/FirebaseModule.kt` — FirestoreSource, StorageSource as singletons

**Tests:**
20. `test/.../ReportRepositoryImplTest.kt`
21. `test/.../EvidenceManagerTest.kt`
22. `test/.../ZipGeneratorTest.kt` — verify ZIP structure: `/recording.mp4`, `/screenshots/`, `/log.txt`, `/report.json`
23. `androidTest/.../ReportDaoIntegrationTest.kt` — CRUD round-trip
24. `androidTest/.../SyncWorkerIntegrationTest.kt` — fires when network reconnects (WorkManager test API)

### Key implementation notes
- ZIP structure is fixed: `recording.mp4`, `screenshots/screenshot_N.png`, `log.txt`, `report.json`
- Platform templates (uTest, Testlio, Test.io) are data-driven — store as JSON resources, not hardcoded
- `SyncWorker` must batch writes — never write one field at a time
- Empty state on `ReportListScreen`: show onboarding prompt, never a blank screen or spinner
- `ReportListScreen` must remain performant with 25+ reports — use `LazyColumn`
- `PromptRepository` interface exists now but has no implementation yet — that is correct

### Stop conditions — Sprint 2
- `./gradlew test` passes — 0 failures
- `./gradlew lint` — 0 errors
- ZIP bundle structure verified by unzipping a generated ZIP in a test
- `grep -rn "getFirestore\|collection(" --include="*.kt" | grep -v SyncWorker | grep -v FirestoreSource` returns 0
- `SyncWorker` is the only file containing Firebase write calls
- Empty state renders correctly on `ReportListScreen`

---

## 16. Sprint 3 — Quality Engine

**Scope:** Deterministic quality scoring, coaching hints, severity advice, completeness checking, duplicate detection.

### Session start prompt
```
Read CLAUDE.md. We are in Sprint 3: Quality Engine.
Sprint 2 checklist has been signed off. Build the files in Section 16.
Stop when all stop conditions are met. Do not start Sprint 4.
```

### Files to create — in this order

1. `feature/quality/domain/CoachingHintStrings.kt` — ALL coaching hint text lives here; none in QualityScorer
2. `feature/quality/domain/QualityScorer.kt` — pure function per Section 10; zero IO, zero AI
3. `feature/quality/domain/SeverityAdvisor.kt` — tree logic; returns Severity recommendation from evidence
4. `feature/quality/domain/CompletenessChecker.kt` — detects missing required fields
5. `feature/quality/domain/DuplicateDetector.kt` — similarity check against user's existing reports
6. `feature/quality/domain/ScoreReportUseCase.kt`
7. `feature/quality/data/QualityRepositoryImpl.kt`
8. `feature/quality/ui/QualityScoreViewModel.kt`
9. `feature/quality/ui/QualityScoreScreen.kt` — animated score reveal; upgrade prompt at score <70

**Update OutcomeFeedback recording:**
10. Update `BuildReportUseCase.kt` to record `preAiQualityScore` in `OutcomeFeedbackEntity`
    (the score before AI suggestions are applied — this data point starts accumulating here)

**Tests:**
11. `test/.../QualityScorerTest.kt`
    - Empty report → score = 0
    - Fully complete report → score = 100
    - Same input 1,000 times → all scores equal (determinism)
    - Every sub-score below 80% of max → coaching hint generated
    - Partial evidence → partial credit (not zero)
12. `test/.../SeverityAdvisorTest.kt` — all 5 severity levels × 5 evidence scenarios = 25 test cases minimum
13. `test/.../CompletenessCheckerTest.kt` — each missing field caught independently
14. `test/.../DuplicateDetectorTest.kt` — 10 non-duplicates return no false positives

### Key implementation notes
- `QualityScorer` must have zero `import android.*` lines — enforce with `grep` in the Architecture Gate
- `QualityScorer` must have zero `suspend` functions — it is synchronous and pure
- The upgrade prompt at score <70 must NOT appear when score ≥70 (test both branches)
- Score animation on `QualityScoreScreen` is a UX requirement — not cosmetic; it drives the emotional impact
- Coaching hints display at most 3 at a time; sorted by `scoreImpact` descending
- `CoachingHintStrings.kt` entries must be ≤120 characters — enforce with a string length assertion test

### Stop conditions — Sprint 3
- `./gradlew test` passes — 0 failures
- `./gradlew lint` — 0 errors
- Coverage ≥90% on `feature/quality/domain/` (Jacoco report)
- `QualityScorer` determinism test passes across 1,000 iterations
- `grep -rn "AIService\|suspend\|Flow\|import android" feature/quality/domain/QualityScorer.kt` returns 0
- Coaching hint string length assertion passes (all ≤120 chars)
- `OutcomeFeedbackEntity.preAiQualityScore` is being populated after scoring

---

## 17. Sprint 4 — AI Assistant

**Scope:** AI-assisted title, summary, repro step generation; safety validation; rate limiting.

### Session start prompt
```
Read CLAUDE.md. We are in Sprint 4: AI Assistant.
Sprint 3 checklist has been signed off. Build the files in Section 17.
Before starting: confirm the per-report AI cost at P90 usage from the Sprint 0 Track 4 memo
is under $0.20/report. If it exceeds this, flag it to the human before proceeding.
Stop when all stop conditions are met. Do not start Sprint 5.
```

### Files to create — in this order

1. `core/data/remote/ai/dto/ChatRequest.kt` — OpenAI request DTO (serializable)
2. `core/data/remote/ai/dto/ChatResponse.kt` — OpenAI response DTO (serializable)
3. `core/data/remote/ai/dto/VisionRequest.kt` — vision endpoint request DTO
4. `core/data/remote/ai/DefaultPromptRepository.kt` — implements `PromptRepository`; all prompts here
5. `core/data/remote/ai/OpenAIService.kt` — sole HTTP caller; implements `AIService`
6. `feature/ai/domain/AiRateLimiter.kt` — token-bucket per BuildConfig.AI_RATE_LIMIT_RPM
7. `feature/ai/domain/SafetyValidator.kt` — implements all four failure criteria from Section 9
8. `feature/ai/domain/GenerateTitleUseCase.kt`
9. `feature/ai/domain/GenerateSummaryUseCase.kt`
10. `feature/ai/domain/FormatReproStepsUseCase.kt`
11. `feature/ai/data/AiRepositoryImpl.kt`
12. `feature/ai/ui/AiAssistViewModel.kt`
13. `feature/ai/ui/AiAssistScreen.kt` — AI suggestions visually distinct from tester-written text

**Update OutcomeFeedback recording:**
14. Update relevant UseCase to set `postAiQualityScore` in `OutcomeFeedbackEntity`
    (the score after AI suggestions are applied — captured when tester accepts suggestions)
15. Update UI to set `testerOverrodeSuggestion = true` when tester edits AI output before submitting
16. Update UI to populate `aiSuggestionsAccepted` and `aiSuggestionsRejected` lists

**Tests:**
17. `test/.../SafetyValidatorTest.kt`
    - Criterion (a): reference to absent UI element → `SafetyResult.Blocked`
    - Criterion (b): Critical severity with no crash evidence → `SafetyResult.Blocked`
    - Criterion (c): repro step not in event log → `SafetyResult.Blocked`
    - Criterion (d): device metadata mismatch → `SafetyResult.Blocked`
    - Correct, grounded output → `SafetyResult.Pass`
18. `test/.../DefaultPromptRepositoryTest.kt` — all 4 methods return non-empty strings for any valid ReportContext
19. `test/.../AiRateLimiterTest.kt` — 21 requests in 60s → request 21 returns RateLimitExceeded
20. `test/.../OpenAIServiceTest.kt` — uses MockWebServer; never hits real network in tests

### Key implementation notes
- `OpenAIService` is the ONLY file making HTTP calls — enforce via Architecture Gate grep
- Prompt strings must be in `DefaultPromptRepository` only — not in UseCases, not inline
- `SafetyValidator` must run before displaying ANY AI output — there is no bypass path
- On network error during AI call: fall back to original tester input; never show blank or freeze
- On rate limit hit: show retry countdown in UI; never a silent failure
- AI-generated text must be visually distinct from tester-written text (different background or label)
- Vision calls (screenshot analysis) cost more than text calls — model separately; SafetyValidator
  uses vision only when screenshots are present in the evidence bundle

### Stop conditions — Sprint 4
- `./gradlew test` passes — 0 failures
- `./gradlew lint` — 0 errors
- All 5 SafetyValidator test cases pass (4 BLOCKED + 1 PASS)
- `grep -rn "OkHttp\|HttpClient\|retrofit\|openai" --include="*.kt" | grep -v OpenAIService | grep -v dto` returns 0
- `grep -rn '"gpt-4\|"gpt-5\|"claude' --include="*.kt" | grep -v BuildConfig` returns 0 (no hardcoded model strings)
- `OutcomeFeedbackEntity.postAiQualityScore` is populated after AI suggestions accepted
- `OutcomeFeedbackEntity.testerOverrodeSuggestion` is set correctly when tester edits AI output

---

## 18. Sprint 5 — Analytics, Flywheel & Launch Readiness

**Scope:** Acceptance rate dashboard, practice mode, outcome feedback entry, and all launch gates.
This is the sprint where the flywheel starts generating real training signal.

### Session start prompt
```
Read CLAUDE.md. We are in Sprint 5: Analytics, Flywheel & Launch Readiness.
Sprint 4 checklist has been signed off. Build the files in Section 18.
Stop when all stop conditions and the Launch Readiness Gate are met.
This is the final sprint before Private Alpha.
```

### Files to create — in this order

1. `feature/analytics/domain/RecordOutcomeUseCase.kt` — tester submits ACCEPTED/REJECTED ★ flywheel
2. `feature/analytics/domain/AcceptanceRateUseCase.kt` — computes rate from OutcomeFeedbackDao
3. `feature/analytics/domain/PracticeModeUseCase.kt` — simulated report flow with scoring
4. `feature/analytics/data/AnalyticsRepositoryImpl.kt`
5. `feature/analytics/ui/OutcomeFeedbackScreen.kt` — where tester enters ACCEPTED/REJECTED ★ flywheel
6. `feature/analytics/ui/AnalyticsDashboardViewModel.kt`
7. `feature/analytics/ui/AnalyticsDashboardScreen.kt` — acceptance rate, quality trend, time saved
8. `feature/analytics/ui/PracticeModeScreen.kt`

**Empty states (required on every analytics screen):**
9. Update `AnalyticsDashboardScreen` with empty state for 0 reports — show potential, not a blank
10. Update `ReportListScreen` empty state (refine if needed after Sprint 2)

**Legal screens (must be live before Private Alpha — not negotiable):**
11. `feature/onboarding/ui/ConsentScreen.kt` — tester consent flow from Track 3 legal memo
12. `feature/onboarding/ui/AiTrainingOptInScreen.kt` — separate screen, separate opt-in, separate data path

**Tests:**
13. `test/.../RecordOutcomeUseCaseTest.kt` — submitting feedback correctly links to reportId
14. `test/.../AcceptanceRateUseCaseTest.kt` — rate computed correctly from seeded OutcomeFeedback data
15. `androidTest/.../OutcomeFeedbackEndToEndTest.kt` — full flow: submit report → get outcome → OutcomeFeedbackDao has correct row

### Key implementation notes
- `OutcomeFeedbackScreen` must be reachable from a completed, submitted report — not buried in settings
- `AnalyticsDashboardScreen` empty state must show the POTENTIAL of the chart, not just "No data yet"
- `PracticeModeScreen` should feel like a game — immediate score, "improve this" retry loop — not a tutorial
- Consent screen must appear on first launch before any capture or report activity
- AI training opt-in must be a distinct checkbox with its own confirmation — never bundled
- Firebase Security Rules must be reviewed and locked before launching to Private Alpha testers
- Verify `FLYWHEEL_ENABLED = true` in all build variants before the Launch Readiness Gate

### Stop conditions — Sprint 5
- `./gradlew test` passes — 0 failures
- `./gradlew lint` — 0 errors
- `OutcomeFeedbackDao` has non-zero rows from internal testing (flywheel is recording)
- `grep -rn "getFirestore\|collection(" feature/analytics --include="*.kt"` returns 0 (analytics reads Room only)
- Empty states present on all analytics screens
- Consent flow live and tested on at least one physical device

### Launch Readiness Gate — before first Private Alpha invite is sent
- All five sprint checklists signed off by human (PRD V4.0)
- Crashlytics crash-free rate >99.5% in internal testing (7-day window, ≥100 sessions)
- No open P0 or P1 bugs
- Legal consent flow live and legally reviewed
- Firebase Security Rules locked — no open read/write in production project
- Google Play Closed Testing track submission in progress
- `FLYWHEEL_ENABLED = true` confirmed in release build
- `OutcomeFeedbackEntity` has non-zero rows from internal testing
- `BuildConfig.AI_MODEL_STANDARD ≠ BuildConfig.AI_MODEL_SAFETY` confirmed in build.gradle.kts

---

## 19. Anti-Patterns — NEVER DO THESE

These are architecture violations. Any PR containing these is blocked before merge.

```
1. Firestore called from a ViewModel or UseCase
   Fix: go through RepositoryImpl → SyncWorker

2. Prompt strings outside DefaultPromptRepository.kt
   Fix: move to DefaultPromptRepository; changing a prompt must touch exactly one file

3. Runtime permission requested inside a Composable
   Fix: use PermissionManager exclusively; Screen composables call ViewModel, not permission APIs

4. Build.MANUFACTURER / MIUI version checks outside OemCompat.kt
   Fix: add the check to OemCompat; call OemCompat from feature code

5. OutcomeFeedbackEntity absent from the Sprint 1 Room schema
   Fix: it is non-negotiable — add it before any other entity

6. Capture logic or MediaProjection on the main thread
   Fix: CaptureService is a ForegroundService; all capture runs in its own coroutine scope

7. AI output displayed to the tester before SafetyValidator has returned Pass
   Fix: no bypass path exists; every AI result flows through SafetyValidator

8. Hardcoded model strings ("gpt-4.1", "gpt-4.1-mini")
   Fix: BuildConfig.AI_MODEL_STANDARD and BuildConfig.AI_MODEL_SAFETY only

9. Room mocked in an integration test
   Fix: use Room.inMemoryDatabaseBuilder — that is what the Room testing library provides

10. A feature package importing another feature package
    Fix: move the shared type to core/domain/model

11. FLYWHEEL_ENABLED = false in any non-test environment
    Fix: OutcomeFeedback recording is always on in production; it is the company's core asset

12. A new Room entity added without a Migration object
    Fix: entity changes require a Migration from version N to N+1; exportSchema catches missed ones

13. AI Safety Layer bypassed "just for testing"
    Fix: add a test double for SafetyValidator that returns Pass; never skip the validator itself
```

---

## 20. Build & Run Commands

```bash
# Development
./gradlew assembleDebug
./gradlew installDebug

# Testing
./gradlew test                            # Unit tests (no device required)
./gradlew connectedAndroidTest            # Device integration tests (device required)
./gradlew jacocoTestReport                # Coverage → build/reports/jacoco/

# Quality
./gradlew lint                            # Zero errors required before sprint sign-off
./gradlew detekt                          # Static analysis

# Room schema export (run after any entity change; commit generated JSON)
./gradlew :app:kaptDebugKotlin

# Release build
./gradlew assembleRelease                 # Requires signing config in local.properties

# Version checks
./gradlew --version                       # JVM version must be 17.x
java -version                             # Must match
```

**Windows PowerShell note:** Use `.\gradlew` instead of `./gradlew`.
**Windows Command Prompt note:** Use `gradlew` (no prefix at all).

---

## 21. Git Commit Conventions

```
Pattern: type(scope): short description in sentence case

Types:   feat | fix | test | refactor | chore | docs | perf
Scopes:  capture | report | quality | ai | analytics | core | sync | ux | ci | flywheel

Examples:
  feat(capture): add MediaProjection foreground service with OEM compat
  feat(flywheel): add OutcomeFeedbackEntity to Room schema version 1
  fix(bubble): resolve MIUI overlay z-index offset on Android 14
  test(quality): add QualityScorer determinism test across 1000 iterations
  refactor(ai): move all prompt strings to DefaultPromptRepository
  chore(deps): bump Compose BOM to 2026.06.00
  docs(claude-md): update Kotlin version to 2.1.20
```

---

## 22. Environment Variables & BuildConfig

```
# local.properties — NEVER committed to version control
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=org-...

# Signing config (add when preparing release build)
KEYSTORE_PATH=...
KEYSTORE_PASSWORD=...
KEY_ALIAS=...
KEY_PASSWORD=...
```

```
# app/build.gradle.kts — defaultConfig block
buildConfigField("String", "AI_MODEL_STANDARD", "\"gpt-4.1-mini\"")
buildConfigField("String", "AI_MODEL_SAFETY", "\"gpt-4.1\"")
buildConfigField("int",    "QUALITY_SCORE_UPGRADE", "70")
buildConfigField("int",    "AI_RATE_LIMIT_RPM", "20")
buildConfigField("int",    "SYNC_INTERVAL_MINUTES", "30")
buildConfigField("boolean","FLYWHEEL_ENABLED", "true")
```

```
# Firebase — two separate projects
google-services-debug.json   → /app/   (for local development)
google-services.json         → /app/   (production — never committed)

# Both filenames in .gitignore
```

---

## 23. Flywheel Instrumentation — The Company's Core Asset

Every report submitted through BugReel must capture these data points, starting from
the first report in Private Alpha. This is not optional and not deferrable.

```
At report creation (Sprint 1+):
  ✓ reportId links all subsequent OutcomeFeedback entries

At quality score generation (Sprint 3+):
  ✓ preAiQualityScore — score before any AI suggestions are shown

At AI suggestion acceptance (Sprint 4+):
  ✓ postAiQualityScore — score after AI suggestions are applied
  ✓ testerOverrodeSuggestion — true if tester edited any AI output
  ✓ aiSuggestionsAccepted — list of field names tester kept unchanged
  ✓ aiSuggestionsRejected — list of field names tester overwrote

At tester outcome entry (Sprint 5+):
  ✓ platformOutcome — ACCEPTED / REJECTED / PENDING
  ✓ rejectionReason — tester enters this when marking REJECTED
  ✓ reportedByTesterAt — timestamp of outcome entry
```

The graduation from the static heuristic `QualityScorer` to a trained ML model happens when
the accumulated `OutcomeFeedback` dataset shows the trained model outperforms the static
rubric on a held-out validation set by ≥15%. Until that threshold is crossed, the static
`QualityScorer` runs. The data collection starts at Sprint 1. The model training happens post-launch.
Do not conflate "data collection" with "model training" — they are separate timelines.

---

## 24. Living Document Protocol — Verify Before Every Sprint

This file decays. Before each sprint session, a human verifies these three items
and updates the table in Section 2 if anything has changed.

| Item | Command | Pass if |
|---|---|---|
| Android Studio version | Check Help → About inside Android Studio | Shows Quail 1 or later |
| Compose BOM version | Check developer.android.com/jetpack/compose/bom | Latest BOM in Dependencies.kt |
| Kotlin version | Check kotlinlang.org/docs/releases.html | Latest stable in Dependencies.kt |

If any version has changed, update `buildSrc/Dependencies.kt` and this file before running
the sprint session. Claude Code will use whatever is in `Dependencies.kt` — stale versions
generate stale code.

**CLAUDE.md must be updated when:**
- A sprint is completed and signed off (mark it COMPLETE in the section header)
- A tech stack version changes
- An architectural decision is revised
- A new anti-pattern is discovered during device testing
- A sprint stop condition changes based on PRD V4.0 checklist outcomes

Do not treat this file as immutable. It is a living document. The last line of every sprint
session should be: "Is CLAUDE.md still accurate? Update if not, then commit."

---

*Last updated: July 2026 · BugReel Pro V4.0 build-ready · Sprint 0 complete*
