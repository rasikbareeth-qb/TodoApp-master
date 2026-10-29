# iOS Code Review Guidelines

Generic baseline rules used by the `ios-code-reviewer` agent. Extend or override per team.

## Severity levels
- **Critical** – crash, data loss, security/privacy leak, broken core flow.
- **High** – incorrect behavior, memory leak, threading bug, unhandled error on a user path.
- **Medium** – maintainability, performance or accessibility issue likely to bite later.
- **Low** – style, naming, minor cleanup.

## 1. Correctness & safety
- No force unwraps (`!`), `try!` or `as!` unless the invariant is guaranteed and commented.
- Handle all `throws`/`Result` paths; no silently swallowed errors (`try?` must be intentional).
- `switch` over enums should be exhaustive; avoid `default` hiding new cases.
- Guard index access on arrays/collections; validate external data (EventKit, network, user input).
- Avoid `fatalError` in production paths unless truly unreachable.

## 2. Memory management
- Closures stored on `self` (callbacks, handlers, Combine sinks, timers, notifications) use `[weak self]`.
- Delegates are `weak`.
- No retain cycles between view controllers, cells and their configurations.
- Remove observers / invalidate timers where required.

## 3. Concurrency
- UI updates happen on the main thread / `@MainActor`.
- `Task {}` usage: cancellation considered, no unstructured tasks leaking `self`.
- Shared mutable state is protected (actor, serial queue, or main-actor isolation).
- No blocking calls (sync I/O, semaphores) on the main thread.
- Continuations (`withCheckedContinuation`) resumed exactly once.

## 4. UIKit
- Respect view controller lifecycle; no heavy work in `viewDidLoad`/`cellForItemAt`.
- Diffable data source snapshots applied consistently; item identifiers are stable and `Hashable`.
- Cells are configured fully on reuse (no stale state).
- Use Auto Layout correctly; no ambiguous or conflicting constraints, `translatesAutoresizingMaskIntoConstraints = false` for code-built views.
- Storyboard identifiers/outlets match code.

## 5. SwiftUI (when present)
- Correct property wrappers (`@State` private and owned, `@StateObject` vs `@ObservedObject`, `@Binding`).
- Keep `body` cheap and side-effect free; use `.task`/`.onAppear` for work.
- Stable `id`s in `ForEach`.

## 6. Privacy & security
- Required `Info.plist` usage descriptions exist for every permission (e.g. Reminders, Calendars).
- Authorization status checked and denial handled gracefully.
- No secrets, tokens or personal data hard-coded or logged.
- Sensitive data stored in Keychain, not `UserDefaults`.

## 7. Architecture & maintainability
- Single responsibility: view controllers don't own persistence/business logic.
- Extensions split by concern are fine; keep files focused and reasonably sized.
- Avoid duplicated logic; prefer small reusable helpers.
- Access control is as tight as possible (`private`, `fileprivate`, `final`).
- Dead code, commented-out code and stray `print` statements removed.

## 8. Naming & style (Swift API Design Guidelines)
- Types `UpperCamelCase`, members `lowerCamelCase`, clear at the call site.
- Booleans read as assertions (`isComplete`, `hasDueDate`).
- Prefer `let` over `var`, value types over reference types where appropriate.
- Consistent formatting and indentation; no magic numbers/strings — use constants or assets.

## 9. Performance
- Avoid repeated expensive work (date formatters, fetches) in hot paths; cache formatters.
- Avoid unnecessary full reloads — prefer targeted snapshot updates.

## 10. Accessibility & localization
- Interactive elements have accessibility labels; Dynamic Type supported.
- User-facing strings are localizable (`NSLocalizedString` / `String(localized:)`).

## 11. Testing
- New logic (models, stores, formatting) should have unit tests.
- Flag untestable designs (hard dependencies, singletons) and suggest seams.
