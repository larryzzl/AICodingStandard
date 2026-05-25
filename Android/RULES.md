# RULES.md - Android Architecture Specifications and Red-Lines

This document defines architectural standards, network design, and non-negotiable safety/performance red lines for Android development. All Android code must adhere to these rules.

## 1. Layered Architecture & Strict Dependency Flow

Dependencies flow strictly from top to bottom. Sibling and reverse dependencies are prohibited:

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

### Layer Definitions:
1. **Presentation (UI)**: Jetpack Compose screens, ViewModels, and state/event streams. ViewModels call Domain Use Cases.
2. **Domain (Business)**: Pure Kotlin (no `android.*` or UI dependencies). Contains domain models (Entities), Repository interfaces, and Use Cases.
3. **Data (Infrastructure)**: Implements Domain Repository interfaces. Manages remote APIs (Retrofit), local db (Room), and cache.
4. **Core (Foundation)**: Feature-agnostic utilities, shared design system, and telemetry interfaces.

---

## 2. MVI & Unidirectional Data Flow (UDF)

Every functional screen must implement the standard MVI (Model-View-Intent) pattern:

1. **Immutable State**: `State` must be an immutable Kotlin `data class`, modified only via `copy()`.
2. **Intent-Driven Actions**: Encapsulate all UI events in a single `Intent` sealed class/interface, processed via `processIntent(intent)`.
3. **Lifecycle-Aware Collection**: Collect UI state in Composables using `collectAsStateWithLifecycle()`. Never use `collectAsState()` as it can leak resources in the background.
4. **Single-Pass Side Effects**: Dispatch one-off events (navigation, toasts) via a buffered `Channel` and collect them in the UI using `LaunchedEffect` and `repeatOnLifecycle`.
5. **Declarative UI & Preview Isolation**:
   - **Stateless UI**: Keep Composables stateless by hoisting state. Pass state down and lambdas up.
   - **Preview Isolation**: Do not pass ViewModels or DI components to Composables. Accept only raw state and lambda callbacks to support `@Preview` blocks.
   - **Side-Effect Boundary**: Wrap lifecycle-triggered tasks in `LaunchedEffect` or `DisposableEffect`; never trigger actions directly in recomposition.
   - **List Keying & Stability**: Always use stable keys in lazy lists (`key = { it.id }`). Use `@Stable`/`@Immutable` or Kotlinx `ImmutableList` for list items to avoid unnecessary recomposition.

---

## 3. Module Decoupling and Routing

- **Stateless Modules**: Feature modules must not maintain internal cross-module global states. Shared data (e.g., session info) must reside in a Shared Repository in the Data Layer.
- **Router-Driven Decoupling**: Never reference or instantiate another feature module's Activity or Composable screen directly. Use `Router` or `Navigator` abstractions registered in the DI container.
- **Type-Safe Navigation**: Use Jetpack Navigation 2.8+ type-safe routes (`@Serializable` classes/objects) defined in common navigation contracts.

---

## 4. Android Network & Data Architecture

- **Offline-First**: Repositories must return Room cached data (as a `Flow`) first, then silently fetch from the network in the background and update Room to trigger UI updates.
- **Error Mapping**: Map raw network/I/O exceptions (`IOException`, `HttpException`) to domain-level exceptions in the Data Layer before exposing them.
- **Token Auto-Refresh**: Use OkHttp `Authenticator`/`Interceptor` to handle `401` responses, refresh tokens, and retry requests transparently.
- **Lifecycle Binding**: Bind async flows to lifecycle scopes (e.g., `viewModelScope`) to ensure cancellation when the scope is cleared.
- **Secure Storage**: Store tokens, credentials, and PII only in secure containers:
  - Key-Value: `EncryptedSharedPreferences` backed by KeyStore.
  - Databases: Room DB encrypted with SQLCipher (key stored in KeyStore).
- **DTO Isolation**: Network DTOs (`@SerializedName`/`@SerialName`) must not leak past the Data Layer. Map them to pure Domain Entities in the Repository.
- **Mandatory Pagination**: All list API endpoints must use pagination (cursor/offset). Enforce query limits in Repositories.

---

## 5. Prohibited Practices (Zero-Tolerance Red Lines)

### A. Network Red Lines
- **No Network Requests in UI/Domain**: UI, ViewModels, and UseCases must never invoke Retrofit or OkHttp. Encapsulate all requests in `RemoteDataSource` inside the Data Layer.
- **No Hardcoded Secrets**: Do not hardcode API domains, keys, or secrets. Inject them via `local.properties` and the **Secrets Gradle Plugin**.
- **No Bypassing SSL**: Trusting all certificates (e.g., dummy `X509TrustManager`, returning true in `HostnameVerifier`) is strictly prohibited. Use standard **Network Security Config** for self-signed certificates.
- **No Logging Sensitive Data**: Never log passwords, tokens, PII, or auth headers to Logcat. Enable `HttpLoggingInterceptor` only when `BuildConfig.DEBUG` is true.

### B. Threading & Concurrency
- **No Main Thread Blocking**: High-latency operations (I/O, database, cryptography, parsing) must run on background dispatchers (`Dispatchers.IO` or `Dispatchers.Default`).
- **No Unmanaged Async**: Do not use `GlobalScope` or raw `Thread`s. Enforce structured concurrency using lifecycle-bounded CoroutineScopes.

### C. Layer Pollution
- **No Context Leaks**: Do not pass `Context`, `Activity`, or UI objects to Domain or Data layers. If needed in Data Layer, inject `@ApplicationContext` via DI.
- **No Feature-to-Feature Static Singletons**: Manage cross-module states through Scoped DI, not global singletons.

### D. Security & Data Storage
- **No Plaintext PII/Token Storage**: Storing tokens or PII in standard `SharedPreferences` or plaintext databases/files is prohibited.
- **No Insecure Component Exposure**: Android components in manifest must set `android:exported="false"` unless explicitly exposed with permissions.
- **No Background Preview Leaks**: High-security screens must set `FLAG_SECURE` in host Activities to block screenshots and task manager previews.
- **No Weak Biometric Session Keys**: Biometric access must be bound to a hardware-backed KeyStore key (`BiometricPrompt.CryptoObject`) that invalidates upon new biometric enrollment.

### E. Dependency Injection
- **No Service Locators**: Avoid dynamic DI resolution (e.g. `Koin.get()`, `EntryPoints`) in ViewModels/Composables. Use constructor injection (`@Inject`).

---

## 6. Performance, Power, and Telemetry

### Performance & Power
- **Lazy Initialization**: Initialize heavy SDKs lazily on background threads using the Jetpack **App Startup** library to avoid blocking app startup.
- **Image Loading**: Use `Coil` or `Glide` with LRU caching, auto-downsampling, and lifecycle awareness. Never load raw high-res images directly.
- **Stateless Core UI**: Core design system components must be stateless, contain no business logic, and communicate only via callbacks.
- **Resource Cleanup**: Close all streams, cursors, and sockets immediately using Kotlin's `.use {}`.

### Telemetry Isolation (No Vendor Lock-in)
- Do not call vendor SDKs (e.g. Firebase, Datadog) directly in feature modules. Use `IMonitoringService` from the Core Layer:
```kotlin
monitoringService.trackEvent("payment_success", mapOf("amount" to 99.0))
```
- Implement the concrete adapter for `IMonitoringService` only in the `app` or dedicated telemetry modules.

---

## 7. Dynamic Configuration and i18n

- **Environment Profiles**: Inject profiles (`dev`, `qa`, `prod`) dynamically via Gradle build variants/flavors. Never switch environments manually in code.
- **100% Localization**: Do not hardcode user-facing strings. Retrieve all text from `strings.xml`.

---

## 8. TDD Test Coverage Standards

1. **Domain Layer**: 100% unit test coverage for UseCases/Interactors.
2. **Data Layer**: Full unit test coverage for Repositories and Data Sources. Use `MockWebServer` for network mocking.
3. **Presentation Layer**: 100% unit test coverage for ViewModel state transitions and side effects using `kotlinx-coroutines-test` (`StandardTestDispatcher` and `runTest`).

---

## 9. Implementation Checklist

Verify before and after changes:
- [ ] Layer flow: Presentation -> Domain -> Data -> Core.
- [ ] Composables are stateless (hoist state to parent or ViewModel).
- [ ] ViewModels expose immutable state via `StateFlow` collected via `collectAsStateWithLifecycle()`.
- [ ] Side effects use `Channel` flows collected in lifecycle-aware UI scopes.
- [ ] DTOs do not leak outside the Data Layer (mapped to Domain Entities).
- [ ] Network requests and async operations are lifecycle-bound (e.g., `viewModelScope`).
- [ ] No context/UI references leak to ViewModel, Domain, or Data layers.
- [ ] Sensitive data logging is disabled in production (uses `BuildConfig.DEBUG`).
- [ ] Sensitive data is encrypted (`EncryptedSharedPreferences`/SQLCipher Room DB).
- [ ] Manifest components audit `android:exported` and high-security screens use `FLAG_SECURE`.
- [ ] Strings are localized in `strings.xml`.
- [ ] Streams and resources are closed using `.use {}`.
