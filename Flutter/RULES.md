# RULES.md - Flutter Architecture Specifications and Red-Lines

This document defines architecture, state management, and safety/performance red lines for Flutter/Dart. All code must adhere to these rules.

## 1. Layered Architecture & Strict Dependency Flow

Dependencies flow strictly top-to-bottom: Presentation -> Domain -> Data -> Core.
No sibling/reverse dependencies allowed.

### Folder Structure (Feature-First)
Organize code by feature directories containing localized layers:
```
lib/
  ├── core/                 # Shared utils, design system, telemetry abstractions
  └── features/
      └── authentication/
          ├── domain/       # Pure Dart: Entities, Use Cases, Repo interfaces
          └── presentation/ # Widgets, state managers (Riverpod/BLoC), ViewStates
```

### Layer Definitions:
1. **Presentation (UI)**: Widgets, state managers, view-states, and events.
2. **Domain (Business)**: Pure Dart. No imports of `package:flutter/...` or platform packages (`dart:io`, `dart:html`). Contains Entities, Repo interfaces, and Use Cases.
3. **Data (Infrastructure)**: Implements Repos. Manages APIs (e.g. `Dio`), local DBs (`Isar`, `Hive`, `sqflite`), cache.
4. **Core (Foundation)**: Feature-agnostic utils, shared tokens, telemetry interfaces.

---

## 2. State Management & Unidirectional Data Flow (UDF)

Every screen must implement the UDF pattern:
1. **Immutable State**: State classes must be immutable (`freezed`/`equatable`). Never mutate directly; use `state = state.copyWith(...)`.
2. **Event-Driven Actions**: Encapsulate UI events in a single sealed class/enum. Process via single entry point (e.g. `onEvent(event)`). Never call events/APIs directly in widget `build`.
3. **Single-Pass Side Effects**: Dispatch one-off actions (navigation, dialogs, snackbars) via streams. Do not store single-pass events in UI State.
4. **Framework Rules**:
   - **Riverpod**: Never call `ref.read` inside `build` body; use `ref.watch`. Limit `ref.read` to callbacks and event handlers.
   - **BLoC**: Avoid nested `BlocBuilder`s. Combine states or use `BlocListener` for one-off side-effects.
5. **UI Optimization**:
   - **Stateless UI**: Hoist state to controllers. Pass data down, notify parent via callbacks.
   - **BuildContext Safety**: Never pass `BuildContext` to ViewModels/Repos/UseCases. Check `context.mounted` before use after async gaps.
   - **Rebuild Control**: Use `const` widget constructors. Use specific selectors (e.g. `ref.watch(provider.select(...))`) to prevent rebuilding on unrelated state changes.
   - **List Performance**: Always use `ListView.builder` or `SliverList` with unique `ValueKey(item.id)` for dynamic lists.

---

## 3. Module Decoupling and Routing

- **Stateless Modules**: Features must not maintain internal cross-module global states. Shared data (e.g. sessions) must reside in a Shared Repository in the Data Layer.
- **Router Decoupling**: Never import/instantiate another module's Widgets/screens directly. Navigation must use `go_router` or custom navigator interfaces in DI.
- **Type-Safe Routing**: Use type-safe route paths (`go_router_builder`) to avoid path typos and unsafe dynamic parameters.
- **Routing Parameters**: Pass primitive IDs instead of complex objects. Let target page fetch entities from its repo.
- **Route Guards**: Handle auth or permission redirects at router level via redirect guards, not inside widget UI rendering.

---

## 4. Network & Data Architecture

- **Offline-First**: Repos must return cached DB data first (as a `Stream`/cache), then fetch network in background and update DB.
- **Error Mapping**: Map raw exceptions (`DioException`, `SocketException`) to domain exceptions in the Data Layer before exposing.
- **Token Auto-Refresh**: Use `Dio` Interceptors to catch `401`, refresh tokens, and retry requests transparently.
- **Resource Binding**: Bind stream subscriptions to controller lifecycle and cancel in `dispose()`.
- **Secure Storage**: Store credentials, secrets, and PII only in encrypted containers:
  - Key-Value: `flutter_secure_storage` (Keychain/EncryptedSharedPreferences).
  - Databases: Encrypted databases (Hive with encryption, or SQLCipher with sqflite).
- **DTO Isolation**: Network DTOs (`json_serializable`) must not leak past Data Layer. Map to pure Domain Entities in Repos.
- **Safe Deserialization**: Catch parsing errors on JSON serialization to prevent runtime `TypeError` crashes. Use code-generators with default values.
- **Cross-Platform Safety**: Do not import `dart:io` or `dart:html` directly. Use conditional imports or `kIsWeb` guards.
- **Parallel Requests**: Use `Future.wait` for independent network requests instead of sequential awaits.
- **Mandatory Pagination**: All list API endpoints must use pagination (cursor/offset) with query limits in Repos.

---

## 5. Prohibited Practices (Zero-Tolerance Red Lines)

### A. Network Red Lines
- **No Network Requests in UI/Domain**: UI and UseCases must never invoke HTTP clients. Encapsulate in `RemoteDataSource`.
- **No Hardcoded Secrets**: Inject API domains, keys, and secrets via `--dart-define` / `--dart-define-from-file`.
- **No Unencrypted HTTP**: Enforce HTTPS protocol. Block `http://` URLs via native build configs (Network Security Config on Android / ATS on iOS).
- **No Bypassing SSL**: Overriding `badCertificateCallback` to return `true` or bypassing SSL handshake is prohibited.
- **No Logging Sensitive Data**: Never log passwords, tokens, or PII. Limit detailed logging using `kDebugMode` or `kReleaseMode` checks.

### B. Threading & Concurrency
- **No Main Thread Blocking**: High-latency tasks (JSON decoding, crypto, I/O, scaling) must run off main isolate using `compute()` or `Isolate.spawn()`.
- **No setState After Dispose**: Check `if (mounted)` or `context.mounted` before calling `setState` or updating state in controllers.
- **No Unmanaged Streams**: All streams, controllers, and listeners must be closed/cancelled during disposal.

### C. Layer Pollution
- **No BuildContext / Widget Leaks**: Passing `BuildContext` or Widget references to Domain/Data layers is prohibited.
- **No UI Styling in Models**: Domain models must not import `package:flutter/material.dart` or use styling classes (`Color`, `IconData`, `TextStyle`).
- **No Feature-to-Feature Static Singletons**: Manage cross-module states through DI/providers, not global static singletons.

### D. Security & Data Storage
- **No Plaintext PII/Token Storage**: Storing tokens or PII in standard `shared_preferences` or plaintext databases/files is prohibited.
- **No Secrets in Source Control**: Keystores, Google Services config keys, and `.env` files must be excluded from version control.
- **No Absolute Platform Paths**: Never hardcode paths (e.g. `'/sdcard/'`). Always use helper libraries like `path_provider`.
- **No Background Preview Leaks**: High-security screens must prevent screenshots and system task manager previews using background blurs or secure flags.
- **No Weak Biometrics**: Biometric auth must bind to hardware-backed keys and invalidate on new biometric enrollments.

### E. Dependency Injection
- **No Service Locator in Widgets**: Avoid resolving dependencies via `GetIt.I<T>()` in Widgets. Use Constructor Injection or Provider/Riverpod.

---

## 6. Performance, Power, and Telemetry

### Performance & Power
- **Const Constructors**: Use `const` constructors for widgets/decorations to optimize widget rebuilding.
- **RepaintBoundary**: Wrap complex widgets, animations, or regularly updating sections in `RepaintBoundary` to avoid full repaints.
- **Avoid Opacity Widgets**: Avoid the `Opacity` widget (especially in animations). Use alpha-channel colors or toggle visibility with `Visibility`/`Offstage`.
- **Avoid Raw shrinkWrap on Scrollables**: `shrinkWrap: true` on scrollables (e.g. `ListView`) forces height calculation on all children, defeating virtualization. Use slivers or layout constraints.
- **Minimize Expensive Clipping**: Avoid `Clip.antiAliasWithSaveLayer`. Use `Clip.hardEdge` or `Clip.none` where possible.
- **Lifecycle-Aware Animations**: Pause or stop `AnimationController` and Lottie/Rive animations when widgets go offscreen or in background.
- **Image Loading**: Use `cached_network_image` with LRU cache. Never load raw uncompressed high-res images in memory.
- **Resource Cleanup**: Override `dispose()` to release memory for controllers (animation, text, scroll, streams).

### Telemetry Isolation
- Do not call vendor SDKs (e.g. Firebase, Datadog) directly in features. Use `IMonitoringService` from Core:
  `monitoringService.trackEvent('payment_success', {'amount': 99.0});`
- Implement concrete adapters for `IMonitoringService` only in the `app` or dedicated telemetry modules.

---

## 7. Dynamic Configuration and i18n

- **Environment Profiles**: Inject profiles via Dart defines at compilation. Never switch environments manually in code.
- **100% Localization**: Retrieve all user-facing text from localizations (e.g. `AppLocalizations.of(context)`).
- **RTL Compatibility**: All UI layouts must support RTL. Avoid directional-locked widgets (use `EdgeInsetsDirectional`, `AlignmentDirectional` instead of raw `EdgeInsets` / `Alignment`).
- **SafeArea Constraints**: Wrap screen layouts with `SafeArea` to avoid overlaps with notches/status bars.

---

## 8. TDD Test Coverage Standards

1. **Domain Layer**: 100% unit test coverage for UseCases and Domain Logic.
2. **Data Layer**: Full unit test coverage for Repositories and Data Sources. Use `mockito` or `mocktail`.
3. **Presentation Layer**: 100% unit test coverage for ViewModel state transitions. Golden tests should be used for critical UI components.
4. **Deterministic Golden Testing**: Golden tests must run in a containerized/pinned-SDK environment with fixed screen resolution and target OS.
5. **Mock Cleanup**: Reset or re-initialize mocks in `setUp` and `tearDown` to prevent state leak across tests.

---

## 9. Implementation Checklist

Verify before and after changes:
- [ ] Folder layout is Feature-First (`features/feature_name/presentation|domain|data`).
- [ ] Layer dependency flow: Presentation -> Domain -> Data -> Core.
- [ ] Domain contains pure Dart without references to `flutter/...` or platform packages.
- [ ] State is completely immutable, and mutated only via `copyWith`.
- [ ] Riverpod `ref.read` is never used inside `build` method body.
- [ ] BuildContext is never stored, and checked with `context.mounted` before use after async gaps.
- [ ] Streams, controllers, scroll controllers, and animation controllers are disposed.
- [ ] Network DTOs do not leak outside the Data Layer.
- [ ] Safe JSON deserialization is implemented to prevent runtime `TypeError` crashes.
- [ ] Platform check guards or conditional imports are used to avoid compile-time issues on web/native.
- [ ] Sensitive data logging is disabled in production (uses `kReleaseMode` or checks).
- [ ] Sensitive data and credentials are encrypted (`flutter_secure_storage`).
- [ ] Secrets and `.env` files are excluded from source control (`.gitignore`).
- [ ] UI layouts are RTL-compatible and wrapped with `SafeArea`.
- [ ] Const constructors are utilized wherever possible.
- [ ] No direct `Opacity` widgets are used in animations, and no raw `shrinkWrap: true` on scrollables.
