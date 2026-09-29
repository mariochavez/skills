---
name: simplify-swift
description: >
  Simplifies Swift and SwiftUI code toward plain, value-oriented, main-actor-first Swift
  while preserving exact behavior. Use after writing or changing Swift, when reviewing a
  Swift diff, package or Xcode target, or when asked to simplify, clean up, or make Swift
  more idiomatic.
---

# Swift Simplifier

Refine Swift and SwiftUI for clarity **while preserving exact behavior**: the same UI output, navigation, error messages, ordering and public API.

## Principles

- **Plain SwiftUI is plenty.** Views, `@Observable` models, `@State`, `@Environment` and `async/await` carry most apps. Coordinators, Combine pipelines and protocol layers are paid for when the app already needs them.
- **The one-sitting test.** Open a feature folder; in one pass you know where the state lives, who changes it and which views read it.
- **Values first.** Structs and enums for data; classes for identity (`@Observable` models, AppKit objects).
- **Main actor first, escape on purpose.** UI code and models run on the main actor; heavy work leaves visibly and returns values.
- **Make illegal states unrepresentable.** Enums with associated values replace flag combinations and parallel optionals.
- **Dependencies are visible.** They arrive through initializers or the environment.
- **Obviously correct, not fewest lines.** Remove accidental complexity; keep the necessary kind visible and in one place.

## Process

1. **Scope** — the Swift written or changed this session (`git diff`). A whole-target review happens on request.
2. **Standards** — read `CLAUDE.md`, `AGENTS.md`, `sdd/standards/` and `docs/ARCHITECTURE.md`. A rule stated there wins over this skill.
3. **Toolchain** — before any concurrency change, read the Swift tools version in `Package.swift` and the build settings `SWIFT_VERSION`, `SWIFT_DEFAULT_ACTOR_ISOLATION` and `SWIFT_APPROACHABLE_CONCURRENCY` (section 11). Isolation advice depends on them.
4. **Baseline** — build and test, and record existing warnings and failures, before the first edit:
   ```bash
   swift build && swift test                                    # packages
   xcodebuild -scheme <Scheme> -destination 'platform=macOS' test   # app targets
   ```
5. **Find** — match the code against the index below, and read the `references/patterns.md` section for each match you intend to change.
6. **Gate** — for a whole-target review, or any change that touches public API or visible behavior, list the findings grouped by section and let the user pick. Session-scoped refinements continue straight to step 7.
7. **Refine** in small, behavior-preserving steps.
8. **Verify** — the build has zero new warnings (concurrency warnings included), tests pass, and `swift format lint --recursive .` is clean when the project uses it.
9. **Report** what changed, grouped by section, and what you left alone on purpose.

## Index

Spot the shape on the left, refine toward the target, and read the section in `references/patterns.md` for the reasoning, the cases to leave alone, and an example.

| Spot | Target | § |
|---|---|---|
| `ObservableObject` / `@Published` / `@StateObject` in new code | `@Observable` with `@State`, `@Bindable`, `@Environment` | 1 |
| A ViewModel per view | one model per feature; children take values and bindings | 1 |
| Stored state kept in sync by hand | computed property | 1 |
| `@State` seeded from a parent's value | the value itself, plus `.task(id:)` | 1 |
| A body longer than a screen, computed properties returning large views | extracted subview structs | 2 |
| `AnyView` | `@ViewBuilder`, `if` / `switch` | 2 |
| The whole model passed to a child view | the value or binding it needs | 2 |
| `Task { }` in `onAppear`, Combine for one-shot async | `.task(id:)`, `async/await` | 3 |
| `DispatchQueue` in async code, `Task.detached` by reflex | main-actor isolation, `@concurrent`, task groups | 4 |
| An actor, `@unchecked Sendable` or `nonisolated(unsafe)` quieting the compiler | restructured ownership | 4 |
| `isLoading` + `errorMessage` flags | `enum LoadState` | 5 |
| Force unwraps on input, `try?` hiding a real failure | `guard let`, `throws` | 5 |
| Boolean flags, parallel optionals, ids that are easy to swap | enums, id structs | 6 |
| A protocol with one conformer, or one made for mocking | concrete type plus fixtures | 7 |
| `.shared` singletons reached from views | `@Environment` or initializer injection | 7 |
| Generated binding, C or AppKit types inside views and models | a thin conversion layer | 8 |
| Concatenated UI strings, hand-formatted numbers and dates | String Catalog, `FormatStyle` | 9 |
| XCTest or mocking frameworks in new tests | Swift Testing with fixtures | 10 |
