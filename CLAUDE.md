# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Today" is a UIKit to-do app (iOS 15.2, Swift 5) that stores its tasks in the system **Reminders** app through EventKit. Every create, update and delete goes straight to Reminders. There is no local persistence layer. It is based on Apple's "App Dev Training – UIKit Essentials" tutorial.

## Commands

The project is `Today.xcodeproj` with one target and one scheme, both named `Today`. It has no package manager dependencies and no test target.

```bash
xcodebuild -project Today.xcodeproj -scheme Today -destination 'platform=iOS Simulator,name=iPhone 15' build
```

```bash
swiftlint
```

SwiftLint also runs as a build-phase script and only warns if it is not installed. `.swiftlint.yml` turns off `line_length` and `trailing_whitespace` and turns on `force_unwrapping` and `empty_count`.

## Architecture

- **`Models/ReminderStore`** is a singleton (`ReminderStore.shared`) that wraps `EKEventStore`. It handles authorization (`requestAccess`), `readAll`, `save` (upsert keyed on `calendarItemIdentifier`) and `remove`. Reminders that have no due date are dropped when reading (`TodayError.reminderHasNoDueDate`).
- **Model mapping**: `Reminder` is the app's value type, and its `id` is the EKReminder's `calendarItemIdentifier`. `Reminder+EKReminder.swift` converts EK → Reminder, and `EKReminder+Reminder.swift` (`update(using:in:)`) converts Reminder → EK. Any new field has to be added on both sides.
- **`TodayError`** is the single error enum used across the store and the mapping code.
- **List screen** (`ListViewController/`): `ReminderListViewController` is a `UICollectionViewController` with a diffable data source keyed by `Reminder.ID`. It is split across extensions: `+DataSource` holds snapshots, cell registration and store calls, and `+Actions` holds button and segment handlers. `ReminderListStyle` (today/future/all) filters `filteredReminders`. `ProgressHeaderView` shows the share of reminders that are complete.
- **Detail screen** (`DetailViewController/`): `ReminderViewController` uses a sectioned collection list with view and edit modes, built from the `Section`/`Row` enums. It reports edits back through an `onChange` closure.
- **ContentViews/** holds custom `UIContentView`/`UIContentConfiguration` cells (text field, text view, date picker) that the detail editor uses.
- `UIColor+Today`, `Date+Today` and `CAGradientLayer+ListStyle` hold the styling and date-formatting helpers.

## Code review tooling

- `.claude/agents/ios-code-reviewer.md` is a read-only reviewer subagent. Its review criteria are in `docs/ios-code-review-guidelines.md`.
- `.github/workflows/claude-pr-review.yml` runs a Claude review on every non-draft PR. Setup instructions are in `docs/Claude PR Review Agent - Setup Guide.md`.
