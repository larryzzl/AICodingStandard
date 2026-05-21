# RULES.md - Web Architecture Specifications and Red-Line Guidelines

This document defines the core architectural blueprint, security standards, rendering standards, and non-negotiable red lines for Web projects. All AI-generated code must strictly adhere to these specifications unless a nearer framework-specific rule file explicitly overrides them.

These rules are intentionally framework-agnostic. They apply to React, Vue, Angular, Svelte, and similar Web application stacks. Framework-specific rules may be added in nested rule files when a project standardizes on a concrete framework.

## 1. Layered Architecture & Strict Dependency Flow

Web projects must separate rendering, business logic, data access, and shared infrastructure. Dependencies must flow in one direction. UI code must not reach directly into infrastructure details.

```
  ┌──────────────────────────────────────────────┐
  │              Presentation Layer              │ ◄── Pages, routes, components, view models, UI state
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Application / Domain abstractions
                         ▼
  ┌──────────────────────────────────────────────┐
  │         Application / Domain Layer           │ ◄── Use cases, business rules, domain models, contracts
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core abstractions
                         ▼
  ┌──────────────────────────────────────────────┐
  │                 Data Layer                   │ ◄── API clients, repositories, DTO mapping, persistence
  └──────────────────────┬───────────────────────┘
                         │ Depends ONLY on Core utilities and contracts
                         ▼
  ┌──────────────────────────────────────────────┐
  │                 Core Layer                   │ ◄── Shared primitives, config, logging, telemetry contracts
  └──────────────────────────────────────────────┘
```

- UI components must not directly call network clients, storage APIs, analytics SDKs, or platform-specific infrastructure.
- Business rules must not be embedded inside reusable visual components.
- Remote DTOs must never leak into UI components. Data must be mapped into domain models or view models before reaching the Presentation Layer.
- Cross-feature dependencies must go through explicit public contracts, route APIs, shared domain services, or dependency injection. Direct feature-to-feature imports are prohibited unless explicitly approved by the project architecture.

## 2. Component Design & Unidirectional Data Flow

Every page, route, or independent business unit must follow a predictable unidirectional data flow:

1. **Explicit State Ownership**: Each piece of state must have one clear owner. Shared state must be promoted only when multiple independent consumers genuinely need it.
2. **Stateless Reusable Components**: Reusable components must receive data through props, inputs, or parameters and notify actions through callbacks, events, or output bindings. They must not fetch data, navigate, mutate global state, or call business services directly.
3. **Container vs. Presentational Separation**: Page-level or container components may connect to application services and state stores. Leaf UI components should remain rendering-focused and testable in isolation.
4. **One-Time Side Effects**: Toasts, dialogs, redirects, downloads, focus jumps, and analytics events must be modeled as effects or commands, not as persistent UI state.
5. **Stable Collection Keys**: Dynamic lists must use stable, unique keys or tracking identifiers. Index-based keys are prohibited when list order can change.
6. **No Render-Cycle Side Effects**: Rendering functions, templates, computed values, and derived state must remain pure. Network calls, navigation, storage writes, subscriptions, timers, and analytics calls must run only in appropriate lifecycle or effect boundaries.

## 3. State Management Boundaries

State must be scoped deliberately. Do not turn global state into a dumping ground.

- Server state, such as remote records, cache freshness, loading status, and request errors, must be managed separately from local UI state.
- Use a dedicated server-state mechanism when available, such as query/cache libraries, framework data loaders, or repository-level caching.
- Global client state is allowed only for truly shared application state, such as authenticated user summary, permissions, feature flags, locale, theme, and app shell state.
- Local interaction state, such as open menus, selected tabs, temporary form edits, and hover/focus state, must remain local unless there is a concrete cross-component requirement.
- Derived state must not be duplicated in stores when it can be calculated safely from existing source state.
- Store mutations must be explicit, traceable, and testable. Hidden mutation through imported singletons or mutable module-level variables is prohibited.

## 4. Web Network Architecture Specifications

- **No Network Requests in UI Components**: Components, templates, and views must not directly call `fetch`, `XMLHttpRequest`, `axios`, GraphQL clients, or generated HTTP clients. Network access must be encapsulated in API clients, repositories, services, or framework-approved loaders/actions.
- **Unified Error Mapping**: Raw transport errors, HTTP status codes, and backend error payloads must be mapped into application-level errors before they reach UI code.
- **DTO to Domain/View Model Isolation**: API response DTOs must be transformed at the Data Layer boundary. UI components must consume stable app-facing models, not backend-shaped payloads.
- **Request Cancellation**: Long-running or route-bound requests must be cancellable when the user leaves the page, changes filters, or starts a newer request that supersedes the old one.
- **Authentication Refresh Boundary**: Token refresh, session renewal, and `401 Unauthorized` recovery must be handled by a centralized auth/network layer, not individually inside pages.
- **Pagination and Query Limits**: List endpoints and table views must use pagination, cursoring, infinite loading, or virtualization-appropriate chunking. Unbounded full-table fetches are prohibited for production data.
- **Retry Discipline**: Automatic retries must be bounded, must avoid retry storms, and must not retry non-idempotent operations unless explicitly designed for it.
- **Timeouts**: Production network calls must have explicit timeout behavior or rely on a centrally configured client timeout.

## 5. Security Red Lines

The following practices are strictly prohibited:

### A. Secrets and Configuration

- No hardcoded secrets, API keys, access tokens, refresh tokens, private endpoints, encryption keys, or credentials in source code.
- No real secrets in frontend bundles. Any value shipped to the browser must be treated as public.
- No manual environment switching through hardcoded `production`, `staging`, or `localhost` branches in application code. Environment selection must use build-time or runtime configuration.

### B. Authentication and Storage

- Sensitive authentication material must not be stored in `localStorage` or unprotected persistent browser storage unless a documented security decision explicitly permits it.
- Prefer secure, `HttpOnly`, `Secure`, `SameSite` cookies for browser session tokens when the backend architecture supports them.
- Tokens, credentials, and PII must not be placed in URLs, query strings, referrers, analytics payloads, or browser history.
- Logout must clear all relevant client caches, in-memory auth state, and persisted session-related data.

### C. XSS, Injection, and Unsafe HTML

- Raw HTML rendering APIs, such as `dangerouslySetInnerHTML`, `v-html`, direct `innerHTML`, or equivalent APIs, are prohibited unless content is sanitized by an approved sanitizer at the boundary.
- User-generated content must be escaped or sanitized before rendering.
- Do not build SQL, GraphQL, HTML, CSS, shell commands, or URLs through unsafe string concatenation when structured APIs are available.
- Content Security Policy weakening, such as broad `unsafe-inline` or `unsafe-eval`, is prohibited unless explicitly justified and approved.

### D. Logging and Monitoring

- Do not log passwords, tokens, authorization headers, cookies, PII, payment details, or sensitive request/response bodies.
- Session replay, analytics, and error monitoring tools must mask sensitive input fields and scrub sensitive payloads.
- Debug network logging must not be enabled in production builds.

## 6. Performance & Rendering Standards

- **Bundle Discipline**: Large dependencies, admin-only tools, charts, editors, maps, and third-party SDKs must be lazy-loaded when they are not required for the initial route.
- **Route-Level Splitting**: Applications with multiple routes must support route-level code splitting unless the stack or product constraints make it unnecessary.
- **Image Optimization**: Images must use appropriate dimensions, compression, responsive sources, lazy loading, and explicit width/height or aspect-ratio constraints to prevent layout shift.
- **List Performance**: Large tables, feeds, and grids must use pagination, incremental loading, or virtualization. Rendering thousands of DOM nodes at once is prohibited unless measured and justified.
- **Render Purity**: Expensive computation must not run repeatedly inside render/template paths. Use memoization, computed caching, selectors, or preprocessing when appropriate.
- **Stable Layout**: Components must reserve space for async content, media, and dynamic controls to avoid cumulative layout shift.
- **Main Thread Protection**: Heavy parsing, formatting, image processing, compression, encryption, or data transformation must not block user interaction. Use workers, streaming, chunking, or backend processing when needed.
- **Third-Party Scripts**: Third-party scripts must be loaded intentionally, measured, and isolated. They must not block critical rendering unless required for core functionality.

## 7. Accessibility Standards

Web UI must be usable with keyboard, screen readers, and common assistive technologies.

- Use semantic HTML before custom roles. Buttons must be buttons, links must be links, and headings must reflect document structure.
- All form controls must have accessible labels and clear validation messages.
- Interactive elements must be reachable and operable by keyboard.
- Focus order must be logical. Modals, drawers, menus, and popovers must manage focus correctly.
- Do not remove visible focus indicators without providing an accessible replacement.
- Color must not be the only way to communicate meaning.
- Text and important UI states must meet WCAG AA contrast expectations unless a project-specific accessibility standard is stricter.
- Images that convey meaning must have useful alternative text. Decorative images must be hidden from assistive technologies.
- Loading, error, and success states must be announced accessibly when they affect task completion.

## 8. Forms and Validation

- Validation rules must be centralized or shared between UI and application logic when possible. Do not duplicate inconsistent validation rules across components.
- Client-side validation is for user experience only. Server-side validation remains mandatory for trust boundaries.
- Error messages must be associated with the relevant field and must explain the correction needed.
- Forms must prevent accidental duplicate submissions for non-idempotent actions.
- Sensitive fields must disable inappropriate autocomplete or recording behavior where required by product security.
- File uploads must validate type, size, and count before upload and must rely on server-side validation after upload.
- Form state must not leak sensitive values into logs, analytics, URLs, or persisted stores.

## 9. Configuration, Environments, and Feature Flags

- Environment-specific values must come from approved build-time or runtime configuration systems.
- Frontend-exposed configuration must be classified as public. Secrets must remain on the server.
- Feature flags must be read through a centralized abstraction. Components must not directly call vendor-specific flag SDKs unless they are part of a dedicated adapter.
- Production, staging, QA, and local behavior must be explicit and auditable. Hidden environment inference is prohibited.
- Dead feature flags must be removed after rollout completion.

## 10. Internationalization (i18n) and Localization

- User-facing text literals must not be hardcoded inside reusable components or pages. Use the project localization system.
- Dates, times, numbers, percentages, currencies, addresses, and plural forms must be formatted with locale-aware APIs.
- Do not concatenate translated sentence fragments. Use parameterized translation messages.
- Layouts must tolerate longer translated strings without overlap or truncation that hides critical meaning.
- User-visible timezone behavior must be explicit for date-sensitive workflows.

## 11. Telemetry and Monitoring Isolation

- Business logic and UI modules must not call vendor SDKs directly for analytics, logging, feature flags, or error reporting.
- Use project-level abstractions, such as `MonitoringService`, `AnalyticsService`, `Logger`, or equivalent interfaces.
- Telemetry events must avoid PII and sensitive business data unless explicitly approved and scrubbed.
- Error reporting must sanitize request bodies, headers, cookies, tokens, and user-entered fields.
- Performance metrics should identify route, operation, and high-level outcome without leaking sensitive details.

## 12. Testing Standards

- Pure business logic, validators, mappers, reducers, selectors, and formatters must have unit tests.
- API clients, repositories, DTO mappers, and error mappers must be tested with mocked transport or contract fixtures.
- Component tests must verify behavior and accessible output, not only implementation details.
- Critical flows, such as login, checkout, onboarding, search, submission, and permission-sensitive flows, must have integration or end-to-end coverage.
- Avoid relying on snapshots as the primary assertion for behavior. Snapshots may supplement but must not replace meaningful assertions.
- Tests must cover loading, empty, error, success, permission-denied, and retry states when those states exist.
- New bugs should be reproduced by a failing test before the fix when practical.

## 13. Framework-Specific Addendum Policy

Framework-specific rule files may refine these standards but must not weaken security, accessibility, testing, or data-boundary requirements.

Examples:

```
Web/React/RULES.md
Web/Vue/RULES.md
Web/Angular/RULES.md
```

Framework addendums may define:
- Component patterns and lifecycle rules.
- Approved state management libraries.
- Routing conventions.
- SSR, SSG, and hydration constraints.
- Testing utilities and file naming conventions.
- Framework-specific prohibited APIs.

## 14. Implementation Checklist

Before and after Web implementation changes, verify:

- Layer direction is preserved.
- UI components do not directly call network, storage, analytics, or vendor SDKs.
- Reusable components remain stateless and render-focused.
- DTOs do not leak into UI.
- Server state and client UI state are separated.
- Requests are cancellable where route or user action can supersede them.
- Sensitive data is not logged, persisted insecurely, or exposed in URLs.
- Raw HTML rendering is avoided or sanitized.
- Dynamic lists use stable keys and large lists are paginated or virtualized.
- Images and async content reserve layout space.
- User-facing strings are localized.
- New UI is keyboard accessible and screen-reader aware.
- Relevant unit, component, integration, or end-to-end tests are added or updated.
