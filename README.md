# SwiftUI Architect

An agent skill that designs and maintains modular SwiftUI apps: vertical feature slices as Swift packages, a small shared horizontal layer, and an app target that owns composition and native SwiftUI navigation.

This README is for people. Agents read [SKILL.md](skills/swiftui-architect/SKILL.md) and the files in [references/](skills/swiftui-architect/references/).

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

- Module map and dependency direction: `XCore`, `XUI`, `XLive`, `XTests`; `Core`, `CoreUI`, `DesignSystem`; infrastructure.
- What goes in Core and CoreUI, and when to duplicate instead.
- App composition: the `XApp` initializer as the dependency-injection root, environment keys, `XScreen` composition views.
- Native SwiftUI navigation: feature `XDestination` values, an app-owned `Router`, one mapping file, registration at stack roots, local presentations.
- Feature UI shapes: views, view models, focused models, ownership.
- Design system: tokens, `XStyle` conformances with dot syntax, domain-free components.
- Localization: per-module String Catalogs, generated symbols, generated asset symbols.
- Testing each feature in isolation.
- Naming and formatting conventions.

Out of scope, and covered by other skills:
- SwiftUI view code technique: [SwiftUI Expert](https://github.com/AvdLee/SwiftUI-Agent-Skill), [SwiftUI Pro](https://github.com/twostraws/swiftui-agent-skill)
- Swift Concurrency: [Swift Concurrency](https://github.com/AvdLee/Swift-Concurrency-Agent-Skill), [Swift Concurrency Pro](https://github.com/twostraws/Swift-Concurrency-Agent-Skill)

## Layout

```
swiftui-architect-skill/
├── .claude-plugin/              Claude Code plugin and marketplace manifests
├── skills/swiftui-architect/
│   ├── SKILL.md                 entry point: shape, dependency rules, ownership, workflows
│   └── references/
│       ├── modules.md           targets, folders, Package.swift template, Core/CoreUI, ports
│       ├── composition.md       XApp.swift, dependency injection, environment keys, XScreen
│       ├── navigation.md        XDestination, Router, Destination, registration, presentations
│       ├── feature-ui.md        views, view models, focused models, ownership
│       ├── design-system.md     tokens, styles, components
│       ├── localization.md      String Catalogs, generated symbols, assets
│       ├── testing.md           mocks, stubs, view model and mapping tests
│       └── conventions.md       naming, folders, formatting
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
