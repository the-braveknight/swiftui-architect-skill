---
name: swiftui-architect
description: Architects modular SwiftUI apps as vertical feature-slice Swift packages (XCore, XUI, XLive, XTests) on a shared horizontal layer (DesignSystem, Core, CoreUI), with an app target that owns composition, dependency injection and native SwiftUI navigation. Use when creating or restructuring a SwiftUI app's modules, adding a feature, screen or view model, wiring navigation between screens or features, deciding where a type belongs, injecting services, writing design-system styles, or localizing strings and assets in a multi-package SwiftUI codebase.
---

# SwiftUI Architect

Decides where code lives and who owns it in a production SwiftUI app built from Swift packages. For how to write SwiftUI view code itself, defer to a SwiftUI skill (see [Related skills](#related-skills)).

Names: `X` stands for a feature (`Discover`, `MovieDetail`); a feature's second screen takes its own noun (`MovieCast` → `MovieCastView`, `MovieCastViewModel`, `MovieCastScreen`). A *screen* is one navigable page: the feature exports its view, the app composes it in an `XScreen`. The running example is a movie browser.

## The shape

```
App target              composition root: services, screens, navigation state, cross-feature wiring
│
├── Feature packages    vertical slices, one per capability (Discover, Search, MovieDetail, …)
│     XCore · XUI · XLive · XTests
│
└── Horizontal layer    shared by every feature
      DesignSystem      domain-free tokens, styles, components
      Core              shared domain vocabulary (Foundation only)
      CoreUI            domain-aware shared views
      Networking, …     infrastructure, imported only by XLive, XTests and the app
```

## Dependency direction

Each module may import only what its arrow points at. Nothing points back up, and features never point at each other.

```
App ──────► XCore, XUI, XLive, Core, DesignSystem, infrastructure
XUI ──────► XCore, Core, CoreUI, DesignSystem
XLive ────► XCore, Core, infrastructure
XCore ────► Core
XTests ───► XCore, XUI, XLive, Core, CoreUI, infrastructure
CoreUI ───► Core, DesignSystem
Core, DesignSystem ──► Foundation / SwiftUI only
```

Hard rules:
- A feature never imports another feature. Navigation between features goes through the app; service needs go through a port the consuming feature owns.
- XCore never imports SwiftUI or infrastructure.
- DesignSystem never knows the domain: no feature models, services or destinations.
- Transport types (DTOs, generated API types) never leave XLive. XLive maps them to XCore models.
- The app is the only module that sees every feature, so it is the only place features meet.

Targets, folders, Package.swift template, Core/CoreUI rules: [references/modules.md](references/modules.md).

## Who owns what

| Concern | Owner |
|---|---|
| Clients, services, adapters, dependency injection | App: built in `XApp.init`, injected through the environment |
| Navigation state: selected tab, every stack's path | App: `Router` |
| Building screens: reading services, creating view models | App: one `XScreen` per screen |
| Which screen a destination shows | App: `Destination` + `DestinationView` |
| Where a feature's views can take the user | Feature: `XDestination` |
| Views, view models, strings | Feature: `XUI` |
| Models, errors, service protocol, mock | Feature: `XCore` |
| Live service, DTO mapping, persistence | Feature: `XLive` |
| Sheets, full-screen covers, alerts | The presenting feature view, as `@State` |

## Principles

1. **Features own their screens' views; the app composes and navigates them.** A feature never builds or shows another feature's screen.
2. **Feature views stay plain and native.** They receive a view model, render it, and push values with `NavigationLink(value:)`. No routers, coordinators, navigation closures or service lookups in feature views.
3. **View models load and expose state.** They never navigate, build views, or know other features.
4. **Duplicate before you share.** Move a type to Core or CoreUI only when a second feature needs the same concept and both copies must change together.
5. **Every module builds, previews and tests on its own.** XCore ships a mock of its service so XUI previews and tests need nothing else.
6. **Native SwiftUI, nothing invented.** No custom navigation modifiers, wrapper views around stacks, DI containers or infrastructure (logging, retry) that the app doesn't need.

## Navigation in one page

Native SwiftUI only: `NavigationStack(path:)`, `NavigationLink(value:)`, `.navigationDestination(for:)`.

Feature (XUI) declares everywhere its views can go, in its own vocabulary, carrying IDs:

```swift
public enum MovieDetailDestination: Hashable, Codable, Sendable {
    case movie(id: MovieSummary.ID)
    case cast(movieID: MovieSummary.ID)
    case person(id: Int)
}

NavigationLink(value: MovieDetailDestination.person(id: member.personID)) {
    CastCard(member: member)
}
.buttonStyle(.content)
```

App maps every feature destination to an app screen, once, and registers them at each tab's stack root:

```swift
NavigationStack(path: $router[.discover]) {
    DiscoverView(viewModel: DiscoverViewModel(service: discoverService))
        .navigationDestination(for: DiscoverDestination.self) { destination in
            DestinationView(destination: Destination(destination))
        }
        .navigationDestination(for: MovieDetailDestination.self) { destination in
            DestinationView(destination: Destination(destination))
        }
}
```

Register only at stack roots, never inside a feature screen: a screen that appears twice in one stack would register twice, which SwiftUI rejects. Full model, Router, push-or-present, previews: [references/navigation.md](references/navigation.md).

## Composition in one page

`XApp.swift` builds every client, service and adapter in `init()`, keeps them as `let` properties typed as protocols, and injects them on the `Scene`. App `XScreen` views read them and build view models. No DI container.

```swift
var body: some Scene {
    WindowGroup {
        ContentView()
            .environment(router)
    }
    .environment(\.discoverService, discoverService)
    .environment(\.movieDetailService, movieDetailService)
}
```

App file shape, environment keys, tab-root and pushed screens: [references/composition.md](references/composition.md).

## Feature UI in one page

- `XView` (public) owns its view model as `@State`, applies screen modifiers (`.navigationTitle`, `.task { await viewModel.load() }`) and renders `XStateView`.
- `XStateView` shows the content whenever there is some, the feature's error only when there's nothing to show, and progress otherwise.
- `XContentView` lays out the content in its scroll container, one view per resource or section. Views that only display take values; a view takes the view model only to read state of its own or call it, like a paging footer.
- Each of these is its own file. Sections, rows and cards are separate `View` structs with narrow inputs. Toolbars are `ToolbarContent` structs.
- View models are `@MainActor @Observable final class XViewModel` with `private(set)` state for each resource the screen loads, and they do no data processing of their own: services return what the views show. Loading and pagination, including screens with several resources, follow a proposed shape, not a rule.
- A focused model for one concern of a screen is a noun without the `ViewModel` suffix (`WatchlistMembership`).

View and view-model shapes, ownership, presentations: [references/feature-ui.md](references/feature-ui.md).

## Design system, localization, testing

- **DesignSystem**: tokens under one namespace (`DesignSystem.Colors.Text.primary`), native style protocols with dot-syntax members (`.buttonStyle(.chip)`, `.progressViewStyle(.loadingDots)`), domain-free components. See [references/design-system.md](references/design-system.md).
- **Localization**: one String Catalog per XUI target and one in the app; Xcode generates `LocalizedStringResource` symbols used with dot notation (`Text(.discoverNavigationTitle)`, `Button(.discoverActionRetry, action: retry)`); assets use generated symbols (`Image(.logo)`). See [references/localization.md](references/localization.md).
- **Testing**: each feature package has its own scheme and tests; view model tests use actor stubs; Live tests check DTO mapping. See [references/testing.md](references/testing.md).
- **Naming and formatting**: see [references/conventions.md](references/conventions.md).

## Concurrency

Out of scope here; use a concurrency skill. The architecture assumes:
- Swift 6 language mode everywhere, with `NonisolatedNonsendingByDefault`, `InferIsolatedConformances` and `MemberImportVisibility` in packages.
- Packages have no default actor isolation and no hand-written `nonisolated` workarounds. View models and their tests are explicitly `@MainActor`; services are `Sendable`; stateful mocks and stubs are actors.
- The app target may use MainActor default isolation, since it is only SwiftUI composition.

## Workflows

**Add a feature**

```
- [ ] Create Packages/X with XCore, XUI, XLive, XTests (template in modules.md)
- [ ] XCore: models, XError, XServiceProtocol, MockXService + `.mock`
- [ ] XLive: XService mapping DTOs to XCore models and transport errors to XError
- [ ] XUI: XViewModel, XView, XStateView, XContentView, XDestination, Localizable.xcstrings
- [ ] XTests: view model tests with actor stubs, mapping tests
- [ ] Infrastructure: add any endpoints the live service needs
- [ ] App: add the local package to the Xcode project and link XCore, XUI and XLive to the app target
- [ ] App: add the environment key, build the service in XApp.init, inject it on the Scene
- [ ] App: add XScreen(s), Destination cases + Destination.init(_: XDestination), DestinationView cases
- [ ] App: register XDestination at the root of every stack that can show the feature
- [ ] Build the feature scheme and the app; run the feature's tests
```

**Link one feature to another** (MovieDetail opens Trailers)

```
- [ ] Source feature: add a case to its XDestination in its own words (`.trailers(movieID:)`) and push it with NavigationLink(value:)
- [ ] If the source needs data the target owns (does this movie have trailers?), declare a port in the source's XCore with a mock, and implement it in an app adapter
- [ ] App: map the new case in Destination.init(_: SourceDestination) to the target's app screen
- [ ] App: register the target's XDestination in every stack that can reach it, directly or through other features (usually every tab)
```

**Add a screen to an existing feature**

```
- [ ] XUI: view model + views; add a case to XDestination; push it with NavigationLink(value:)
- [ ] App: XScreen that reads services and builds the view model
- [ ] App: Destination case, Destination.init mapping, DestinationView case
- [ ] If the screen is reachable from a stack that didn't show this feature before, register XDestination there
```

**Decide where a type belongs**

1. Used by one feature? It stays in that feature: XCore for models and contracts, XUI for anything visual or localized, XLive for transport.
2. A second feature needs the same concept and both must change together? Pure model → Core. A view that renders Core types → CoreUI.
3. Only looks alike? Duplicate it.
4. Purely visual and domain-free? DesignSystem.
5. Wiring two features together? The app.

## Related skills

If installed, use these for how to write the code; this skill decides where it lives:
- SwiftUI view code: `swiftui-expert-skill` (https://github.com/AvdLee/SwiftUI-Agent-Skill) or `swiftui-pro` (https://github.com/twostraws/swiftui-agent-skill)
- Swift Concurrency: `swift-concurrency` (https://github.com/AvdLee/Swift-Concurrency-Agent-Skill) or `swift-concurrency-pro` (https://github.com/twostraws/Swift-Concurrency-Agent-Skill)

