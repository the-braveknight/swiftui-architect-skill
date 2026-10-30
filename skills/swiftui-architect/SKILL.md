---
name: swiftui-architect
description: Architects modular SwiftUI apps as one Swift package per feature, split into domain, UI and live-service targets over a small shared layer, with an app target that composes screens, injects services and owns navigation. Use when adding a feature or screen to a SwiftUI app, modularizing an iOS app into Swift packages or SPM modules, drawing module boundaries or dependency direction, designing domain models, services or MVVM view models (loading, errors, pagination, search), wiring navigation, routers, coordinators or dependency injection between features, deciding where a type belongs or what to name it, or reviewing an app's architecture, even when the user doesn't name the architecture.
---

# SwiftUI Architect

Decides where code lives, who owns it, and the shape of the pieces that hold a modular SwiftUI app together. For how to write the SwiftUI, concurrency or test code inside those shapes, use the skills in [Related skills](#related-skills).

`X` stands for a feature (`Movie`) or one of its screens (`MovieCast` → `MovieCastService`, `MovieCastViewModel`, `MovieCastView`, `MovieCastScreen`). A *screen* is one navigable page: its feature exports the view, and the app composes it in an `XScreen`.

An app built another way keeps its structure until the user asks to change it. A change there follows the app's own conventions, and takes a shape from this skill only where the app has none ([moving an app into packages](references/modules.md#moving-an-app-into-packages)).

The examples are one small movie browser on the TMDB API:

| Feature | Screens |
|---|---|
| Discover | The Discover tab: featured movies over paged shelves. A genre's movies, with a search in the genre |
| Movie | A movie's details, with a watchlist button, and its cast |
| Person | A person's details and credits, by department |
| Search | The Search tab, with a filter sheet |
| Watchlist | The Watchlist tab, with a search through it and a confirmation before removing a movie |
| Account | Signing in, shown over the Watchlist tab |

## The shape

```
App target              composition root: services, screens, routing
│
├── Feature packages    one per capability, never per tab (Discover, Movie, Search, …)
│     XCore     domain models, errors, a service protocol and mock per screen
│     XUI       view models, views, XDestination, strings
│     XLive     live services: API responses → domain models
│     XTests    view model and mapping tests
│
└── Packages under the features
      Shared        Shared: domain vocabulary several features use (Foundation only)
                    SharedUI: views several features show
                    SharedLive: API mappings and client steps several features share
      DesignSystem  domain-free tokens, styles and components
      Networking    the API's types and client, generated from an OpenAPI document when there is one
```

A feature owns every screen of its capability, including the root screen of every tab that shows it, and whatever those screens share. A tab's root screen is named for its tab (`DiscoverView`), anything else for what it is (`GenreDetailView`, `CastMemberCard`) ([inside a feature](references/modules.md#inside-a-feature)).

## Dependency direction

Each module imports only what its arrow points at. Nothing points back up, and features never point at each other.

```
App ────────► XCore, XUI, XLive, Shared, SharedUI, DesignSystem, Networking
XUI ────────► XCore, Shared, SharedUI, DesignSystem
XLive ──────► XCore, Shared, SharedLive, Networking
XCore ──────► Shared
XTests ─────► XCore, XUI, XLive, Shared, SharedUI, Networking
SharedUI ───► Shared, DesignSystem
SharedLive ─► Shared, Networking
Shared, DesignSystem, Networking ──► Apple frameworks and their own dependencies only
```

Hard rules:
- A feature never imports another feature. The app is the only module that sees every feature, so features meet only there: links through its destination mapping, data through ports it implements.
- XCore never imports SwiftUI or Networking.
- Generated API types never leave the Live targets (XLive, SharedLive). Live services map them to domain models, and transport errors to the feature's errors.
- DesignSystem knows no domain: no feature models, services or destinations.

## Who owns what

| Concern | Owner |
|---|---|
| What several features use: models, views, API mappings and client steps, the places they open (`SharedDestination`) | `Shared`, `SharedUI`, `SharedLive` |
| State several features read or change, such as the watchlist or a signed-in account | The one feature that owns it; the others reach it through ports |
| Building the client and services, injecting them through environment keys | App: `<AppName>App.init` |
| Building a screen: reading services, creating its view model | App: one `XScreen` per screen ([composition](references/composition.md)) |
| Navigation state, and which screen each destination shows | App: `Router`, `Destination`, `DestinationView` |
| What a screen presents, the value a presentation is about, and a sheet's draft | The screen's view model, as plain `var`s its views bind |

## Principles

1. **Features own their screens; the app composes and routes them.**
2. **Models speak the app's language.** Each is shaped for what a screen shows and can't hold an illogical value; the API's shape stops at the Live targets.
3. **Services do the work; view models hold state.** A service returns view-ready domain values; a view model assigns them and tracks loading, errors and paging. Nothing sits between a view model and its sources: no repositories, use cases or managers.
4. **Feature views stay plain.** They render a view model, push values with `NavigationLink(value:)`, and take from the app only the screens they present: no routers, navigation closures or service lookups.
5. **Duplicate before sharing.** Something moves to the shared package only when a second feature needs the same concept and both copies must change together.
6. **Every module builds, previews and tests on its own**, XUI on XCore's mocks.
7. **Native SwiftUI and Apple's frameworks, nothing invented.** No navigation libraries, DI containers or wrapper views around stacks.
8. **One screen per kind of content.** Things that load, page and search alike share a screen; when they differ, each gets its own, rather than one screen that switches on what it was given.

## The shapes in brief

- **Models and errors.** Domain models are `Equatable`, `Sendable` values (`Movie`, and `MovieSummary` for a list). Each feature has its own error enum, cased by what its screens show (`offline`, `notFound`, `unavailable`); an empty answer is never an error. [domain](references/domain.md)
- **Services.** A protocol per screen in XCore returns view-ready domain values (`Page<Item>` for a paged list) and throws the feature's error. XLive implements it over the networking client; XCore ships its mock (`.mock`). [domain](references/domain.md)
- **View models.** `@MainActor @Observable public final class XViewModel`, one per screen, built by the app with IDs and services. Each resource has its data, `isLoading…` and `…Error`; a paged one adds `next…Page`, `isLoadingNext…Page` and `next…PageError`. A part only some views read, with its own input or loading, is a nested `@Observable` model, never a `…ViewModel`. [view models](references/view-models.md)
- **Views.** `XView` owns the view model as its only `@State`, puts it in the environment and starts the loads. `XPhaseView` shows the content whenever there is data, otherwise the error, progress or the empty state. `XContentView` lays out the values it's given. [views](references/views.md)
- **Navigation.** A feature lists where its views go in its `XDestination`; the places several features open are `SharedDestination` cases. The app maps each case to a screen and registers each type at the root of every stack that can show it. A screen shown as a sheet or cover is a case too, presented by `XView` from content the app passes in. [navigation](references/navigation.md)
- **Concurrency.** Packages build in Swift 6 mode. View models are `@MainActor`, services `Sendable`, live service methods `@concurrent`, and stateful services and mocks actors.

## Gotchas

- Register `navigationDestination` only at stack roots. Inside a feature screen it registers again when that screen appears twice in one stack, and SwiftUI reports a duplicate.
- `XScreen` builds a view model on every `body` pass and `XView` keeps the first, so a view model's `init` only assigns.
- A string literal in a package looks in the app's bundle; use the catalog's generated symbols.
- Never set `.defaultIsolation(MainActor.self)` in a package: generated OpenAPI code doesn't build under it, and domain types must stay nonisolated.
- A type named like one the SDK declares (`Collection`, `Category`, `Section`) is ambiguous wherever both are imported; qualify it from the domain (`MovieCollection`).

## Workflows

Copy the checklist for the task into the response and check items off as they're done.

**Add a feature**

```
- [ ] Check the capability is new: a screen showing what an existing feature owns joins that feature (Add a screen)
- [ ] Create Packages/X, named for the capability, with XCore, XUI, XLive, XTests (template in modules.md)
- [ ] XCore: domain models, XError, and per screen a service protocol and its mock + `.mock`
- [ ] Networking: a client method for each endpoint the feature calls, in openapi.yaml first when the client is generated
- [ ] XLive: a live service per protocol, mapping responses to the domain and the client's error to XError
- [ ] XUI: XViewModel, XView, XPhaseView, XContentView, XDestination, Localizable.xcstrings; push places another feature already links to as `SharedDestination`
- [ ] XTests: the screen's own rules, with actor stubs, and mapping tests (view-models.md#testing-view-models)
- [ ] App: add the package to the Xcode project and link XCore, XUI and XLive to the app target
- [ ] App: for each service protocol, an environment key, the service built in <AppName>App.init and injected on the Scene
- [ ] App: a pushed XScreen per pushed screen, Destination cases + Destination.init(_: XDestination), DestinationView cases
- [ ] App, for a tab's root screen: a TabDestination case, its Tab in ContentView, and a tab XScreen that owns the tab's stack and registers SharedDestination and every XDestination it can show
- [ ] App: register XDestination at the root of every stack that can show the feature
- [ ] Build the feature's scheme and the app, and run the feature's tests; fix and repeat until both pass
```

**Add a screen to an existing feature**

```
- [ ] XCore: the screen's service protocol and mock; XLive: its live service
- [ ] XUI: the screen's folder with its view model and views; a case in XDestination, pushed with NavigationLink(value:)
- [ ] XUI: a section another of the feature's screens already shows becomes one view in Components/ both use, named for what it is
- [ ] App: environment key, the service built in <AppName>App.init and injected
- [ ] App: XScreen, Destination case and mapping, DestinationView case
- [ ] If a stack that didn't show this feature can now reach it, register XDestination there
- [ ] Build the feature's scheme and the app, and run the feature's tests; fix and repeat until both pass
```

**Modularize an existing app**

```
- [ ] Map the app: its capabilities (a feature each), their screens, and what two features already share
- [ ] Networking, then DesignSystem, then the shared package; build the app after each
- [ ] One feature at a time, the one using the fewest others first, as in Add a feature; its reach into other features becomes destinations and ports
- [ ] As screens leave the app target, their services move into <AppName>App.init and their navigation into Router, Destination and DestinationView
- [ ] Build the app and run every package's tests after each step; fix and repeat until they pass
```

**Add search**

- Across the app (a Search tab) → its own screen, usually its own feature.
- Through what a screen already shows → XViewModel keeps the matches of the field's `text`; nothing new loads ([filtering](references/view-models.md#filtering-loaded-values)).
- On the server, within one screen → a mode: a `SearchSession` nested in XViewModel ([modes](references/view-models.md#modes)).

**Link one feature to another** (Movie opens Person)

```
- [ ] If the place is a `SharedDestination`, push it and stop here. If another feature's XDestination already has it, move that case to `SharedDestination`, with its mapping, and push that
- [ ] Source feature: a case in its XDestination in its own words (`.person(id:)`), pushed with NavigationLink(value:), or presented by its XView when the design shows it over the screen (navigation.md#presenting-a-screen)
- [ ] If the source needs data the target owns (is this movie in the watchlist?), a port in the source's XCore with a mock, implemented by an app adapter
- [ ] App: map the new case in Destination.init(_: SourceDestination) to the target's screen; for a presented case, the source's XScreen passes DestinationView to its XView
- [ ] App: register the target's XDestination in every stack that can reach it, usually every tab
- [ ] Build both features' schemes and the app; fix and repeat until they pass
```

**Review an app's architecture**

```
- [ ] Run the boundary checks (modules.md#checking-the-boundaries)
- [ ] Each feature is one capability owning all its screens; the shared package holds nothing only one feature uses
- [ ] Each screen: one view model, built by its XScreen, over its own service protocol and ports
- [ ] Report each finding with its file, the rule it breaks and the fix; change code only when the user asks
```

**Decide where a type belongs**

1. One feature uses it, from one screen or several → that feature: XCore for models and contracts, XUI for anything visual or localized, XLive for mapping.
2. A second feature needs the same concept and both must change together → Shared for a model, SharedUI for a view, SharedLive for a mapping to a Shared model or a client step both call.
3. Only looks alike → duplicate it.
4. Visual and domain-free → DesignSystem. Talks to the API → Networking, mapped in the Live targets. Wires two features together → the app.

## References

Read the reference for the part at hand:

| Reference | Read when |
|---|---|
| [references/modules.md](references/modules.md) | Creating a package or its `Package.swift`; placing and naming what a feature's screens share; a feature, the shared package or DesignSystem; strings, assets and fonts; tests; boundary checks; an app built another way |
| [references/domain.md](references/domain.md) | Domain models, errors, service protocols, live services, mocks and ports, and their names; state several features share |
| [references/networking.md](references/networking.md) | Adding an endpoint; the API client and its errors |
| [references/view-models.md](references/view-models.md) | A view model: loading, several resources, paging, keyed resources, filtering, nested models and modes, tests |
| [references/views.md](references/views.md) | A screen's views: the `XView` family, error and empty states, paging footers, modes, passing data, previews |
| [references/composition.md](references/composition.md) | The app target: `<AppName>App`, environment keys, `ContentView`, `XScreen` |
| [references/navigation.md](references/navigation.md) | Destinations, linking features, navigating in code, push or present, sheets and alerts |
| [references/conventions.md](references/conventions.md) | Naming a type, file or folder |

## Related skills

Load these, when installed, for the code inside the shapes:

| Writing or reviewing | Skill |
|---|---|
| View bodies, state and data flow, modifiers, layout, redraw performance, lists, navigation and presentation APIs, previews | A SwiftUI skill: `swiftui-specialist` (Apple's, in Xcode's coding assistant), `swiftui-expert-skill` or `swiftui-pro` |
| Service and client methods, the bodies of view model loads and their guards against stale results, task groups, cancellation, actors, `Sendable`, test stubs | A Swift Concurrency skill: `swift-concurrency` or `swift-concurrency-pro` |
| View model and mapping tests: suites, `#expect` and `#require`, parameterized tests, traits, async tests | A Swift Testing skill: `swift-testing-expert` |

Where this skill states a shape, such as a view model's state or `XView`'s `.task`, follow it, and use those skills for everything inside it.
