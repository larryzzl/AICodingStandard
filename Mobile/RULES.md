# RULES.md - Mobile Architecture Specifications and Red-Line Guidelines

This document defines the core architectural blueprint, network design standards, and absolutely non-negotiable **safety/performance red lines** for this project. All AI-generated code must strictly and unconditionally adhere to these specifications.

## 1. Layered Architecture & Strict Dependency Flow

The project is divided into four highly cohesive, loosely coupled layers. Dependencies must flow strictly from top to bottom. Lateral (sibling) feature-to-feature direct dependencies and reverse dependencies are strictly prohibited:

```
  ┌──────────────────────────────────────────────┐
  │         Presentation Layer (UI/UX)           │ ◄── Contains Views, ViewModels, Intents, States, Routers
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Domain
                         ▼
  ┌──────────────────────────────────────────────┐
  │         Domain / Business Logic Layer        │ ◄── Pure logic, contains Entities, Use Cases, Repo Interfaces
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core
                         ▼
  ┌──────────────────────────────────────────────┐
  │                 Data Layer                   │ ◄── Implements Repo Interfaces, manages API, DB, Cache
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core
                         ▼
  ┌──────────────────────────────────────────────┐
  │            Core Layer (Foundation)           │ ◄── Utils, Common UI Components, Telemetry Abstractions
  └──────────────────────────────────────────────┘
```

## 2. MVI & Unidirectional Data Flow (UDF) Specifications

Every functional screen or independent business unit must implement the standard MVI pattern:

1. **Immutable State**: `State` must be a completely immutable data class or struct. Modifying the state must be achieved only by creating a new copy (e.g., using Kotlin’s `copy()` or Swift’s `updating()` extensions).
2. **Intent-Driven**: All user actions and event triggers from the View to the ViewModel must be encapsulated into a single unified `Intent` sealed class/enum (e.g., `SendOtpIntent`, `RetryFetchIntent`) and dispatched through a single entry-point method (e.g., `process(intent)`).
3. **Single-Time Side Effects (Effects/Events)**: One-off UI behaviors such as dialog popups, navigation, or toasts must not pollute the persistent `State`. They must be emitted and consumed via single-pass event streams (e.g., Kotlin `SharedFlow`/`Channel` or Swift `PassthroughSubject`).
4. **Declarative UI State Hoisting & Preview Isolation**:
   - **Stateless Views/Composables**: Child views must be kept stateless by hoisting state up to parent containers or the ViewModel. They should receive data via immutable state properties and bubble actions back up using lambda callbacks or closures.
   - **ViewModel Preview Isolation**: Never pass parent `ViewModel` or `Coordinator` instances directly into child or reusable UI components. UI components must accept only localized state models and action lambdas/closures to ensure they can be previewed (e.g., Compose Previews, SwiftUI Previews) and tested in isolation.
   - **Side-Effect Boundary**: Do not trigger business operations or side effects directly within the UI rendering cycle. Any lifecycle-triggered tasks must be bound to native declarative scopes (e.g., `LaunchedEffect` in Compose, `.task` / `.onAppear` in SwiftUI).
   - **List Recomposition & Keying**: When displaying dynamic collections (e.g., in a lazy list or grid), stable, unique keys must be explicitly provided for each item (e.g., `key = { item.id }` in Jetpack Compose or `ForEach(items, id: \.id)` in SwiftUI). This prevents unnecessary recompositions/re-renders of the entire list.

## 3. Module Decoupling and Routing Mechanism

- **Stateless Modules**: Feature modules must not maintain global, cross-module singleton states internally. Shared business data (such as user session info) must reside in a Shared Repository managed within the Data Layer.
- **Router-Driven Decoupling**: It is strictly forbidden to instantiate or reference another feature module's View directly inside a feature's codebase. All cross-module navigation must be routed through `Router` or `Coordinator` abstractions registered in the Dependency Injection (DI) container.

## 4. Mobile Network Architecture Specifications

- **Offline-First with Smart Caching**: When retrieving data, the Repository should immediately return cached local data (if available) and concurrently trigger a silent background network request. Once the network response arrives, it should update the local database and notify the UI to refresh (Stale-While-Revalidate pattern).
- **Unified Error Mapping**: Raw OkHttp errors, SocketTimeout exceptions, or raw `HTTP 500` status codes must never be thrown directly to the Domain or UI layers. The Data Layer must intercept and map them into domain-level exceptions (e.g., `NetworkException.NoConnection`, `NetworkException.ServerMaintenance`).
- **Seamless Token Auto-Refresh**: Intercept `401 Unauthorized` errors inside a network interceptor, pause pending requests, fetch a new Access Token using the Refresh Token, and automatically replay the original request transparently without exposing this flow to the upper business layers.
- **Lifecycle Binding**: All network requests (Coroutines Flow/Task/Combine) must be bound to the caller's lifecycle scope (e.g., `viewModelScope`). When a View is destroyed, a ViewModel is cleared, or a page is closed, all corresponding pending requests must be cancelled immediately to prevent memory leaks.
- **Secure Local Storage & Cryptography**: All sensitive authentication tokens (Access, Refresh, ID tokens), credentials, and Personally Identifiable Information (PII) must be stored exclusively in native secure containers (Keychain for iOS, EncryptedSharedPreferences / KeyStore for Android). If databases (e.g., Room, SQLite) store sensitive cached data, they must be encrypted at rest (e.g., utilizing SQLCipher).
- **DTO to Domain Entity Isolation**: Remote API data models (DTOs) must never leak beyond the Data Layer. The Repository must map network DTOs to pure, UI-agnostic Domain Entities before delivering them to the Domain Layer.
- **Mandatory Pagination & Optimization**: All list-retrieval remote endpoints must use pagination (cursor-based or offset-based). Repositories must enforce query limits, and ViewModels must manage current pages gracefully to prevent high memory usage.
- **Structured Async Exception Boundaries**: All asynchronous execution blocks must enforce strict error-boundary policies. Kotlin Coroutines must utilize a `CoroutineExceptionHandler` or map results to monadic `Result` wraps. Swift Tasks must handle potential exceptions via explicit `try/catch` scopes to guarantee the app never silently crashes from unhandled async exceptions.

## 5. Absolutely Prohibited Practices (Zero-Tolerance Red Lines)

To ensure application stability, security, and performance, the following practices are strictly prohibited:

### A. Network Red Lines

- **No Network Requests in UI or Domain Layers**: ViewModels, UseCases, and Views are absolutely forbidden from directly invoking HttpClient instances (e.g., OkHttpClient, URLSession). All network requests must be encapsulated within `RemoteDataSource` implementations inside the Data Layer.
- **No Hardcoded Sensitive Information**: Absolutely no API domains, port numbers, AppSecrets, API Keys, salting keys, or certificates may be hardcoded in the codebase. These parameters must be retrieved dynamically from generated config classes.
- **No Bypassing of SSL Validation**: Writing code that trusts all certificates (e.g., Bypass SSL Pinning, TrustAll) is strictly prohibited under any circumstances (including local development and staging environments). Legitimate self-signed certificates must be handled safely via the system's native configurations (such as Android's `Network Security Config` or iOS's `App Transport Security`).
- **No Logging of Sensitive Data (PII & Auth)**: Do not log sensitive user data (e.g., passwords, phone numbers, Auth Tokens, network headers, or payloads) to the console or log management systems. All verbose network logging must be restricted to `BuildConfig.DEBUG` (Android) or `DEBUG` compilation flags (iOS).

### B. Threading & Concurrency Red Lines

- **No Main Thread Blocking**: High-latency tasks such as JSON parsing, database reads/writes, file I/O, image scaling, and network requests must never block the main thread. Developers must explicitly specify background dispatchers (e.g., `Dispatchers.IO` or `Task.detached`).
- **No Unmanaged Background Tasks**: Running asynchronous tasks within unmanaged scopes (such as Kotlin's `GlobalScope` or detached, non-lifecycle-aware threads) is strictly prohibited. Structured concurrency must be enforced at all times.

### C. Layer Pollution Red Lines

- **No Feature-to-Feature Static Global Singletons**: Cross-module states must have their lifecycles and scopes strictly managed by the DI container (Scoped DI) to ensure proper memory reclamation.
- **No Platform UI Framework & Context Leaks**: Passing platform-specific context or UI objects (such as Android `Context`, `View`, `Intent` or iOS `UIView`, `SwiftUI.View`) into the Domain or Data Layers is strictly prohibited. This prevents memory leaks and preserves platform-agnostic business logic.

### D. Security & Data Storage Red Lines

- **No Plaintext Token or PII Storage**: Storing sensitive tokens (Auth tokens, Refresh tokens), passwords, or PII in plaintext in unprotected local database tables, standard Shared Preferences, or standard User Defaults is strictly prohibited. Logs cannot contain any PII data or auth information.
- **No ViewModel Leakage in Sub-components**: Passing ViewModel or Router instances directly to child Views, Composables, or cells is strictly prohibited.
- **No Unprotected Sensitive Input Fields**: Inputting sensitive credentials (passwords, PINs, bank details) without disabling OS keyboard dictionary caching is strictly prohibited.
- **No Background Exposure of Sensitive Screens**: High-security screens (authentication, checkout, identity verification) must implement screen masking (e.g., `FLAG_SECURE` on Android, view blurring inside `sceneWillResignActive` on iOS) to prevent OS multitasking previews, screenshots, or screen recordings from leaking sensitive data.
- **No Weak Biometric Session Keys**: Securing local data session accesses via biometrics without binding them to a hardware-backed cryptographic key (e.g., Android Keystore, iOS Keychain) that invalidates upon new biometric enrollment is strictly prohibited.

### E. Dependency Injection Red Lines

- **No Direct Service Locator References**: Referencing the DI container directly inside Views or ViewModels to resolve dependencies dynamically (e.g., using `KoinJavaComponent.get()` or a global `ServiceLocator.resolve()` dynamically) is prohibited. Constructor injection must be utilized to declare all required dependencies explicitly.

## 6. Performance, Power, and Telemetry Monitoring Specifications

### Performance & Power Optimization

- **Lazy Initialization**: Initialization of heavyweight SDKs and intensive resource loading must be executed lazily on background threads, ensuring they never block the application startup path.
- **Image & Memory Management**: Images rendered in lists must utilize an image loader equipped with LRU caching and auto-downsampling capabilities. Direct loading of raw, high-resolution source images into memory is strictly prohibited.
- **Stateless Design System Abstractions**: All shared UI components in the `Core` layer must be entirely decoupling-oriented (stateless). They are strictly forbidden from executing business logic, containing dependencies, navigating to features, or reading global application states. They must only render primitives and notify via closures.

### Telemetry Isolation (Zero Vendor Lock-in)

- To enable seamless switching between third-party monitoring platforms (such as Firebase, Datadog, etc.) in the future, **never** make direct SDK calls (e.g., `FirebaseAnalytics.logEvent()`) inside business or UI modules.
- Utilize the unified monitoring abstraction `IMonitoringService` defined in the Core Layer:

```
// Compliant: Business modules interact only with the abstraction
monitoringService.trackEvent("payment_success", mapOf("amount" to 99.0))
monitoringService.logException(networkException)
```

- Concrete adapter implementations of this SDK must reside strictly in the main `app` module or dedicated telemetry adapter modules within the Data Layer.

## 7. Dynamic Configuration and Internationalization (i18n)

- **Dynamic Environment Configuration**: Configuration files must be split strictly into two sets: `config-nonprod.json` (for Integration, QA, and Staging) and `config-prod.json` (for Production). These must be dynamically injected during compilation via Gradle Tasks or Xcode Schemes. Manual, hardcoded environment switching is forbidden.
- **100% Localization**: Hardcoding user-facing text literals inside UI components is strictly prohibited. All localized copy must be retrieved dynamically from localized resource files (e.g., Android's `strings.xml` or iOS's `Localizable.strings`).

## 8. Design System, Theme, Accessibility, and UI Automation

### 8.1 Dynamic Design Tokens

- All UI styling must be consumed through semantic design tokens.
- Raw colors, font sizes, spacing, radius, shadows, and asset names must not be hardcoded in feature UI.
- Runtime token changes must be propagated through an injected ThemeProvider / ThemeStore / Environment object, not through static globals.

### 8.2 Light, Dark, and High-Contrast Modes

- All semantic color tokens must define light and dark values.
- Components must consume semantic tokens, not raw palette values.
- The design system layer is responsible for resolving token values based on system appearance, user preference, partner configuration, and accessibility settings.
- Shared components and critical screens must be verified in light mode, dark mode, and large text mode.

### 8.3 Accessibility

- All interactive controls must expose meaningful accessibility labels.
- Decorative images must be hidden from accessibility services.
- Informational images must provide localized accessibility descriptions.
- Custom controls must expose proper accessibility roles/traits.
- Text must support Dynamic Type / font scaling.
- Layouts must remain usable under large accessibility text sizes.
- Color must not be the only signal for status, validation, or errors.
- Error, loading, empty, disabled, and success states must be accessible.
- Sensitive information must not be unnecessarily exposed through accessibility labels.

### 8.4 UI Automation Testability

- All key UI elements must expose stable automation identifiers.
- Automation identifiers must not be generated from localized visible text.
- Automation identifiers must not contain PII, tokens, account numbers, phone numbers, or user-specific sensitive data.
- Reusable UI components must allow parent features to provide automation identifiers.
- Dynamic lists must use stable unique keys and stable automation identifiers.
- Automation identifiers should follow a consistent convention:
  `<feature>.<screen>.<element>.<state?>`.

### 8.5 Shared UI Component Contract

Shared UI components may receive:

- Immutable display state.
- Semantic design tokens.
- Localized text or localization keys.
- Accessibility metadata.
- Automation identifiers.
- Event callbacks.

Shared UI components must not:

- Call network APIs.
- Access repositories.
- Resolve dependencies from DI containers.
- Navigate directly to feature modules.
- Read global session state.
- Log analytics directly.
- Own business state.

## 9. TDD Test Coverage Standards

1. **Domain Layer**: All UseCases/Interactors must achieve **100% unit test coverage**.
2. **Data Layer**: All Repositories and Error Mappers must be covered by comprehensive unit tests.
3. **Presentation Layer**: State transition logic and Side-Effect emissions within the ViewModel must achieve **100% unit test coverage**.

## 10. Implementation Checklist

<!-- reason: This mobile-specific checklist belongs with architecture and safety rules, not with generic agent behavior instructions. -->

Before and after mobile implementation changes, verify:

- Layer direction is preserved.
- UI remains stateless where required.
- ViewModels expose immutable state and one-off effects separately.
- DTOs do not leak outside Data.
- Async work is lifecycle-bound.
- Sensitive data is not logged or stored insecurely.
- User-facing strings are localized.
