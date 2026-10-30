# SwiftUI Architect

An agent skill that designs and maintains modular SwiftUI apps: vertical feature slices as Swift packages, a small shared horizontal layer, and an app target that owns composition and native SwiftUI navigation.

This README is for people. Agents read [SKILL.md](skills/swiftui-architect/SKILL.md) and the files in [references/](skills/swiftui-architect/references/). Agents changing the skill follow [CLAUDE.md](CLAUDE.md).

## Why modular

Large SwiftUI codebases are commonly split into many local Swift packages, one per feature plus a few shared layers. Teams converge on this shape for practical reasons:

- **Boundaries the compiler enforces.** A feature can only use what its package declares as a dependency. "Search must not reach into Watchlist" stops being a review comment and becomes a build error.
- **Ownership.** A feature package has a clear owner, a clear public API, and a place for everything it needs. Several teams can work in parallel with few merge conflicts.
- **Faster builds and previews.** Changing one feature rebuilds that package and what depends on it, not the whole app. Previews compile only the feature being previewed.
- **Testability in isolation.** A feature builds and tests against a mock of its own service contract, without the app, the network or other features.
- **Replaceable layers.** Networking, persistence or the design system can change behind a stable contract. A feature can be rewritten, removed or put behind a flag without touching others.
- **Readable architecture.** The folder tree and the dependency graph show how the app is built. New engineers and agents find where code belongs without reading all of it.

## Why use it for production apps

Production apps live for years, grow features, and change hands. The costs of this structure are paid once and stay small; the costs of a monolith grow with every feature:

| Concern | Monolithic target | This architecture |
|---|---|---|
| Accidental coupling | Any file can use any type | Only declared dependencies compile |
| Build time | Grows with the whole app | Scoped to changed packages |
| Testing a screen | Needs app setup and real services | Needs the feature's mock |
| Navigation between features | Scattered pushes and shared routers | One app-owned Router, one mapping file |
| Localization | One huge catalog | One catalog per feature, compile-time-checked symbols |
| Onboarding | Read the codebase | Read the module map |

The trade-offs are real: more `Package.swift` files to maintain, some deliberate duplication instead of premature sharing, and an app target that does the wiring. The skill keeps those costs predictable with templates and strict rules about where code goes.

## What the skill covers

- The module map and dependency direction: feature packages (`XCore`, `XUI`, `XLive`, `XTests`), one per capability whichever tabs show it, over a shared package (`Shared`, `SharedUI`, `SharedLive`), `DesignSystem` and `Networking`, and what goes in each.
- Domain-driven models that describe what the app shows, not what the API sends, with live services mapping API responses and errors into them.
- Networking with Apple's swift-openapi-generator: a generated client wrapped in the app's own, with failures and cancellation classified once.
- Service design and naming, following the Swift API Design Guidelines: a protocol per screen with no repository or use case layers, reads as noun phrases, mocks for previews and tests, consumer-owned ports between features, and one owner for state several features share, such as a signed-in account.
- View models: per-resource loading state, several resources, pagination, resources keyed by what the user picks or types, a search through what's already loaded, nested models for parts of a screen such as a search within it, feature error types, cancellation, and which tests are worth writing.
- The views a screen is made of: `XView`, `XStateView`, `XContentView`, sections and paging footers, and state views that read the data first, so a refresh doesn't redraw them.
- App composition: the `<AppName>App` initializer as the dependency-injection root, environment keys, tab and pushed `XScreen` views.
- Routing: feature `XDestination` values, SharedUI's `SharedDestination` for the places several features open, an app-owned `Router`, one mapping file, registration at stack roots, and presentations held as view model state, down to the value an alert or sheet is about.
- Localization, assets and fonts per module: String Catalogs with generated symbols, asset catalogs with generated symbols, fonts that DesignSystem registers.
- Naming and folder conventions, including what a feature's screens share and what spans the whole feature.

Out of scope, and covered by other skills the agent loads alongside this one:
- SwiftUI view code technique: Apple's `swiftui-specialist`, built into Xcode's coding assistant; [SwiftUI Expert](https://github.com/AvdLee/SwiftUI-Agent-Skill); [SwiftUI Pro](https://github.com/twostraws/swiftui-agent-skill)
- Swift Concurrency: [Swift Concurrency](https://github.com/AvdLee/Swift-Concurrency-Agent-Skill), [Swift Concurrency Pro](https://github.com/twostraws/Swift-Concurrency-Agent-Skill)
- Swift Testing: [Swift Testing Expert](https://github.com/AvdLee/Swift-Testing-Agent-Skill)

## Layout

```
swiftui-architect-skill/
├── .claude-plugin/              Claude Code plugin and marketplace manifests
├── skills/swiftui-architect/
│   ├── SKILL.md                 entry point: shape, rules, gotchas, workflows, which reference to read
│   └── references/
│       ├── modules.md           targets, naming inside a feature, folders, Package.swift, the shared package, DesignSystem, strings, tests
│       ├── domain.md            domain models, errors, service protocols and naming, live services, mocks, ports
│       ├── networking.md        swift-openapi-generator, the API client, errors and cancellation
│       ├── view-models.md       loading state, several resources, pagination, keyed resources, filtering, nested models, tests
│       ├── views.md             XView, XStateView, XContentView, paging footer, passing data
│       ├── composition.md       <AppName>App.swift, dependency injection, environment keys, XScreen
│       ├── navigation.md        XDestination, Router, Destination, registration, push or present
│       └── conventions.md       naming and folders
├── CLAUDE.md                    instructions for agents changing the skill
├── README.md
└── LICENSE
```

## Installation

**Claude Code plugin**

```
/plugin marketplace add the-braveknight/swiftui-architect-skill
/plugin install swiftui-architect@swiftui-architect-skill
```

**skills CLI**

```bash
npx skills add https://github.com/the-braveknight/swiftui-architect-skill --skill swiftui-architect
```

**Manual**

Copy `skills/swiftui-architect/` to `~/.claude/skills/swiftui-architect/` (all your projects) or to `.claude/skills/swiftui-architect/` in a repository (shared with the team).

Claude Code loads the skill when a task matches its description, or on request with `/swiftui-architect`.

## License

MIT. See [LICENSE](LICENSE).
