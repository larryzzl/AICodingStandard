# RULES.md - Android Architecture Specifications and Red-Line Guidelines

This document defines the core architectural blueprint, network design standards, and absolutely non-negotiable **safety/performance red lines** for Android application development within this project. All AI-generated and human-written Android code must strictly and unconditionally adhere to these specifications.

## 1. Layered Architecture & Strict Dependency Flow

The project is divided into four highly cohesive, loosely coupled layers. Dependencies must flow strictly from top to bottom. Lateral (sibling) feature-to-feature direct dependencies and reverse dependencies are strictly prohibited:

```
  ┌──────────────────────────────────────────────┐
  │         Presentation Layer (UI/UX)           │ ◄── Views (Composables/Fragments), ViewModels, ViewStates, Intents
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Domain Layer
                         ▼
  ┌──────────────────────────────────────────────┐
  │         Domain / Business Logic Layer        │ ◄── Pure Kotlin: Entities, Use Cases (Interactors), Repo Interfaces
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core Layer (Platform-agnostic boundaries)
                         ▼
  ┌──────────────────────────────────────────────┐
  │                 Data Layer                   │ ◄── Repo Impls, Data Sources (Room DB, Retrofit API, Cache)
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core Layer
                         ▼
  ┌──────────────────────────────────────────────┐
  │            Core Layer (Foundation)           │ ◄── Utils, Shared Design System, Telemetry Abstractions
  └──────────────────────────────────────────────┘
```

### Layer Definition Details:
1. **Presentation Layer**: Built on top of **Jetpack Compose**. Contains UI logic, Composable screens, ViewModels, and state/event streams. ViewModels communicate with the Domain Layer via Use Cases.
2. **Domain Layer**: The heart of the application. Written in **pure Kotlin** (no dependencies on `android.*` or UI packages). It contains business logic, domain models (Entities), and Repository definitions (interfaces).
3. **Data Layer**: Implements Repository interfaces defined in the Domain Layer. Orchestrates remote calls via **Retrofit/OkHttp**, local persistence via **Room**, and device-level states.
4. **Core Layer**: Provides foundational, cross-cutting concerns (e.g., cryptographic helper classes, localization abstractions, and telemetry interfaces) without feature-specific logic.

---

## 2. MVI & Unidirectional Data Flow (UDF) Specifications

Every functional screen or independent business unit must implement the standard MVI (Model-View-Intent) pattern:

1. **Immutable State**: `State` must be a completely immutable Kotlin `data class`. Modifying the state is strictly allowed only by generating a new instance using the `copy()` function.
2. **Intent-Driven Actions**: All user actions and event triggers from the UI to the ViewModel must be encapsulated within a single unified `Intent` sealed interface or sealed class (e.g., `LoginIntent.Submit`, `LoginIntent.Retry`) and dispatched through a single entry-point method (e.g., `processIntent(intent)`).
3. **Lifecycle-Aware State Collection**: In Composable screens, the `StateFlow` exposed by the ViewModel must be collected using `collectAsStateWithLifecycle()` (from the `androidx.lifecycle.lifecycle-runtime-compose` library). Do **not** use `collectAsState()`, as it does not stop resource flow when the app is in the background, risking memory leaks or background CPU drain.
4. **Single-Pass Side Effects**: One-off events (e.g., navigation, Toast notifications, dialog popups) must not pollute the permanent `State`. They must be dispatched via a Kotlin `Channel(Channel.BUFFERED)` and exposed as a `Flow`, collected in the UI lifecycle-aware scope using `LaunchedEffect` paired with `repeatOnLifecycle`.
5. **Declarative UI State Hoisting & Preview Isolation**:
   - **Stateless Composables**: UI components must be kept stateless by hoisting state up to parent containers or ViewModels. They should receive data via immutable state models and propagate interactions back up using lambda callbacks.
   - **Preview Isolation**: Never pass parent `ViewModel` or DI dependency instances directly into child or reusable Composable functions. All Composables must accept only raw state values and event lambdas to enable isolated renderings in Android Studio `@Preview` blocks.
   - **Side-Effect Boundary**: Do not trigger business operations or state changes directly within the Composable's recomposition block. Any lifecycle-triggered tasks must be wrapped in `LaunchedEffect` or `DisposableEffect`.
   - **List Recomposition & Keying**: When displaying dynamic collections in `LazyColumn` or `LazyRow`, stable and unique keys must be explicitly provided (e.g., `items(items, key = { it.id })`). Ensure data models passed to lists are marked with `@Stable` or `@Immutable`, or utilize Kotlinx `ImmutableList` to prevent whole-list recomposition when only one item changes.

---

## 3. Module Decoupling and Routing Mechanism

- **Stateless Modules**: Feature modules must not maintain global, cross-module singleton states internally. Shared business data (such as user session info) must reside in a Shared Repository managed within the Data Layer.
- **Router-Driven Decoupling**: It is strictly forbidden to instantiate or reference another feature module's Activity or Composable screen directly inside a feature's codebase. All cross-module navigation must be routed through `Router` or `Navigator` abstractions registered in the Dependency Injection (DI) container.
- **Type-Safe Navigation**: If using Jetpack Navigation (2.8+), routes must be defined using Kotlin Serialization type-safe routes (`@Serializable` objects/classes) mapped via interfaces inside common navigation contracts.

---

## 4. Android Network & Data Architecture Specifications

- **Offline-First with Smart Caching**: The Repository should immediately return local cached data from Room (as a reactive `Flow`), concurrently triggering a silent background network request. Once the network response arrives, the Data Layer updates the local database, and Room automatically propagates the updated data to the UI.
- **Unified Error Mapping**: Raw network/I/O exceptions (e.g., `IOException`, Retrofit `HttpException`, `SocketTimeoutException`) must never leak to the Domain or UI layers. The Data Layer must intercept these and map them to domain-level exceptions (e.g., `NetworkError.NoConnection`, `NetworkError.ServerDown`).
- **Token Auto-Refresh**: Authenticated API requests must automatically refresh expired access tokens transparently. Implement this using an OkHttp `Authenticator` or `Interceptor` that intercepts `401 Unauthorized` responses, blocks the queue, calls the token-refresh endpoint, saves the new tokens, and retries the original request.
- **Lifecycle Binding**: All network requests and asynchronous flows must be bound to the caller's lifecycle scope. In ViewModels, use `viewModelScope`. When the ViewModel is cleared, all corresponding pending requests must be cancelled automatically.
- **Secure Local Storage**: Sensitive authentication tokens, credentials, and Personally Identifiable Information (PII) must be stored exclusively in native secure containers:
  - Key-value data: **EncryptedSharedPreferences** backed by the **Android KeyStore**.
  - Structured database: **Room DB** encrypted at rest using **SQLCipher**, with the key securely generated and retrieved from the Android KeyStore.
- **DTO to Domain Entity Isolation**: Remote API data models (DTOs) annotated with `@SerializedName` or `@SerialName` must never leak beyond the Data Layer. The Repository must map network DTOs to pure, UI-agnostic Domain Entities before delivering them to the Domain Layer.
- **Mandatory Pagination & Optimization**: All list-retrieval remote endpoints must use pagination (cursor-based or offset-based). Repositories must enforce query limits, and ViewModels must manage current pages gracefully to prevent high memory usage.

---

## 5. Absolutely Prohibited Practices (Zero-Tolerance Red Lines)

To ensure application stability, security, and performance, the following practices are strictly prohibited:

### A. Network Red Lines

- **No Network Requests in UI or Domain Layers**: ViewModels, UseCases, and Composables are absolutely forbidden from directly invoking Retrofit services or HttpClient instances (e.g., OkHttpClient). All network requests must be encapsulated within `RemoteDataSource` implementations inside the Data Layer.
- **No Hardcoded Sensitive Information**: Absolutely no API domains, port numbers, AppSecrets, API Keys, salting keys, or certificates may be hardcoded in the codebase or standard `strings.xml`. These parameters must be retrieved dynamically from generated config classes or build config parameters injected via `local.properties` + the **Secrets Gradle Plugin**.
- **No Bypassing of SSL Validation**: Writing code that trusts all certificates (e.g., implementing a dummy `X509TrustManager` that performs no checks or returning `true` in `HostnameVerifier`) is strictly prohibited under any circumstances (including local development and staging environments). Legitimate self-signed certificates must be handled safely via the system's native **Network Security Config** (`network_security_config.xml`).
- **No Logging of Sensitive Data (PII & Auth)**: Do not log sensitive user data (e.g., passwords, phone numbers, Auth Tokens, network headers, or payloads) to Logcat or log management systems. Verbose network logging via OkHttp's `HttpLoggingInterceptor` must be wrapped to execute only when `BuildConfig.DEBUG` is true.

### B. Threading & Concurrency Red Lines

- **No Main Thread Blocking (ANR Prevention)**: High-latency tasks such as JSON parsing, database reads/writes, file I/O, cryptographic operations, and network requests must never block the main thread. Developers must explicitly offload these operations to background dispatchers (e.g., `Dispatchers.IO` for I/O, `Dispatchers.Default` for CPU-bound computations).
- **No Unmanaged Background Tasks**: Running asynchronous tasks within unmanaged scopes (such as Kotlin's `GlobalScope` or raw, non-lifecycle-aware `Thread` instances) is strictly prohibited. Structured concurrency must be enforced at all times using `CoroutineScope` bounded to lifecycles.

### C. Layer Pollution Red Lines

- **No Context Leaks**: Passing or storing platform-specific context or UI objects (such as `Context`, `Activity`, `Fragment`, `View`, `Lifecycle`, or Compose states) into the Domain or Data Layers is strictly prohibited. This is a primary source of memory leaks. If a context is required in the Data Layer (e.g., to read files or access system services), the `ApplicationContext` must be injected via Dependency Injection (e.g., Hilt's `@ApplicationContext` annotation).
- **No Feature-to-Feature Static Global Singletons**: Cross-module states must have their lifecycles and scopes strictly managed by the DI container (Scoped DI) to ensure proper memory reclamation.

### D. Security & Data Storage Red Lines

- **No Plaintext Token or PII Storage**: Storing sensitive tokens (Auth tokens, Refresh tokens), passwords, or PII in plaintext in unprotected local database tables, standard `SharedPreferences`, or raw local files is strictly prohibited.
- **No Insecure Component Exposure**: Declaring `<activity>`, `<service>`, `<receiver>`, or `<provider>` with `android:exported="true"` in the `AndroidManifest.xml` without proper permission requirements or intent-filtering protection is strictly prohibited. If a component is for internal application use only, it must be explicitly configured as `android:exported="false"`.
- **No Background Exposure of Sensitive Screens**: High-security screens (authentication, checkout, identity verification) must prevent OS multitasking previews, screenshots, or screen recordings from leaking sensitive data. Implement this by applying `WindowManager.LayoutParams.FLAG_SECURE` in the host Activity’s `onCreate()`, or dynamically toggling it.
- **No Weak Biometric Session Keys**: Securing local data session accesses via biometrics without binding them to a hardware-backed cryptographic key in the **Android KeyStore** (using `BiometricPrompt.CryptoObject`) that invalidates upon new biometric enrollment is strictly prohibited.

### E. Dependency Injection Red Lines

- **No Direct Service Locator References**: Referencing the DI container directly inside Composable functions or ViewModels to resolve dependencies dynamically (e.g., using a global `Koin.get()` or manually pulling from Hilt EntryPoints) is prohibited. **Constructor injection** (using `@Inject` in Hilt or constructor declaration in Koin) must be utilized to declare all required dependencies explicitly.

---

## 6. Performance, Power, and Telemetry Monitoring Specifications

### Performance & Power Optimization

- **Lazy Initialization & App Startup**: Initialization of heavyweight SDKs (e.g., Firebase, crashlytics, telemetry) and intensive resource loading must be executed lazily on background threads, ensuring they never block the application startup path. Use the Jetpack **App Startup** library to coordinate dependencies.
- **Image & Memory Management**: Images rendered in lists must utilize an image loader (preferably **Coil** or Glide) equipped with LRU caching, auto-downsampling capabilities, and lifecycle awareness. Direct loading of raw, high-resolution source images into memory is strictly prohibited.
- **Stateless Design System Abstractions**: All shared UI components in the `Core` layer must be entirely decoupling-oriented (stateless). They are strictly forbidden from executing business logic, containing dependencies, navigating to features, or reading global application states. They must only render primitives and notify via closures/lambdas.
- **Resource Clean-up**: Any opened stream, cursor, or socket must be closed immediately after usage using Kotlin’s `.use {}` extension function to prevent file descriptor leaks.

### Telemetry Isolation (Zero Vendor Lock-in)

- To enable seamless switching between third-party monitoring platforms (such as Firebase Analytics, Datadog, Dynatrace, etc.) in the future, **never** make direct SDK calls (e.g., `FirebaseAnalytics.logEvent()`) inside business or UI modules.
- Utilize the unified monitoring abstraction `IMonitoringService` defined in the Core Layer:

```kotlin
// Compliant: Business modules interact only with the abstraction
monitoringService.trackEvent("payment_success", mapOf("amount" to 99.0))
monitoringService.logException(networkException)
```

- Concrete adapter implementations of this service must reside strictly in the main `app` module or dedicated telemetry adapter modules within the Data Layer.

---

## 7. Dynamic Configuration and Internationalization (i18n)

- **Dynamic Environment Configuration**: Configuration files must be split strictly into environment profiles (e.g., `dev`, `qa`, `prod`). These must be dynamically injected during compilation via Gradle **Build Variants**, **productFlavors**, and **buildTypes**. Manual, hardcoded environment switching in application code is forbidden.
- **100% Localization**: Hardcoding user-facing text literals inside UI components or Kotlin files is strictly prohibited. All localized copy must be retrieved dynamically from localized resource files (e.g., `res/values/strings.xml` and language-specific overrides).

---

## 8. TDD Test Coverage Standards

1. **Domain Layer**: All UseCases/Interactors must achieve **100% unit test coverage**.
2. **Data Layer**: All Repositories, Data Sources, and Error Mappers must be covered by comprehensive unit tests. Use mock web servers (e.g., `MockWebServer` by OkHttp) to mock network behaviors.
3. **Presentation Layer**: State transition logic and Side-Effect emissions within the ViewModel must achieve **100% unit test coverage** using `StandardTestDispatcher` and `runTest` from the `kotlinx-coroutines-test` library.

---

## 9. Implementation Checklist

Before and after Android implementation changes, verify:
- [ ] Layer direction is strictly preserved (Presentation -> Domain -> Data -> Core).
- [ ] Composables remain stateless, hoisting state to the parent or ViewModel.
- [ ] ViewModels expose immutable state via `StateFlow` and collect it in UI via `collectAsStateWithLifecycle()`.
- [ ] Single-time side effects are handled via `Channel` flows, collected in UI lifecycle-aware scopes.
- [ ] DTOs do not leak outside the Data Layer (mapped to Domain Entities before leaving Repository).
- [ ] All network requests and async jobs are lifecycle-bound (e.g., using `viewModelScope`).
- [ ] No raw `Context`, `Activity`, or `Fragment` references leak into ViewModels, Use Cases, or Repositories.
- [ ] Sensitive data is not logged in production (`HttpLoggingInterceptor` tied to `BuildConfig.DEBUG`).
- [ ] Sensitive storage uses `EncryptedSharedPreferences` or SQLCipher-encrypted Room databases.
- [ ] Security boundaries: `android:exported` attributes are audited, and high-security screens use `FLAG_SECURE`.
- [ ] User-facing strings are strictly localized in `strings.xml`.
- [ ] All resources (Cursors, streams) are closed safely via `.use {}`.
