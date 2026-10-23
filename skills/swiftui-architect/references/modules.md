# Modules

The app is built from local Swift packages: one per feature, plus a few shared packages every feature stands on. The compiler enforces the boundaries: a module can use only what its package declares.

## Contents
- Feature packages
- Inside a feature
- Folder layout
- Package.swift template
- Shared, SharedUI and SharedLive
- DesignSystem
- Networking and other infrastructure
- Localization, assets and fonts
- Testing

## Feature packages

A feature is a vertical slice around one capability, and it owns that capability's data contract. It owns every screen of the capability, including the root screens of the tabs that show it: Movie owns a movie's details screen and its cast screen, and a Discover feature whose Movies and TV tabs browse one catalog owns both tabs' roots and the screens they open. Screens and tabs never define the boundary. Name the package after the capability, never after a tab or one of its screens: `Discover`, `Search`, `Watchlist`, `Movie`, `Person`, not `MovieDetail` or `Home`.

| Target | Contains | Imports |
|---|---|---|
| `XCore` | Domain models, `XError`, a service protocol and mock per screen, ports | `Shared` |
| `XUI` | View models, views, `XDestination`, `Localizable.xcstrings`, display mappings | `XCore`, `Shared`, `SharedUI`, `DesignSystem` |
| `XLive` | Live services, mapping API responses and errors to the domain, persistence | `XCore`, `Shared`, `SharedLive`, `Networking` |
| `XTests` | View model tests and mapping tests (Swift Testing) | all of the above, plus `Networking` for response fixtures |

- XCore has no SwiftUI and no infrastructure, so the domain stays testable and usable by any UI ([domain](domain.md)).
- XUI depends on XCore's protocols, never on XLive, so it builds, previews and tests on mocks alone.
- XLive is the only feature target that sees the API. A mapping to a Shared model that several features read comes from SharedLive.
- The app links all three library targets: it builds live services from XLive and shows views from XUI ([composition](composition.md)).

## Inside a feature

Names inside a feature say what a thing is, so a screen, model or view keeps its name whichever screens or tabs show it.

| What | Named for | Example |
|---|---|---|
| A tab's root screen, with its view model, service, error and the models only it shows | Its tab | `MoviesView`, `MoviesViewModel`, `MoviesService` |
| Any other screen | What it shows | `GenreDetailView`, `MovieCastView` |
| A model, view or mapping several of the feature's screens use | What it is | `MovieCollection`, `PosterRail`, `CastMemberCard` |
| What the whole feature owns | The feature | `DiscoverDestination`, `DiscoverLayout`, the `discover.` string keys |

- A section two of the feature's screens show is one view in `Views/`, filed by kind (`Cards/`, `Rails/`), taking the values it shows. A frame several screens put around different content, such as a hero banner over a list of rails, is one view that takes that content as view builders; the actions inside still come from each screen's view model ([views](views.md#view-structure-rules)).
- What only this feature uses stays in it, however many of its screens use it. It moves to Shared, SharedUI or SharedLive only when a second feature needs it ([promotion rule](#shared-sharedui-and-sharedlive)).
- A name an imported module already declares takes a qualifier from the domain: `MovieCollection`, not `Collection` (Swift); `DiscoverLayout`, not `Layout` (SwiftUI). Unqualified, it hides the module's type everywhere in the feature, so code there that means the module's type stops building.

## Folder layout

```
Packages/Movie/
├── Package.swift
├── Sources/
│   ├── MovieCore/
│   │   ├── Models/            Movie.swift, CastMember.swift, MovieError.swift
│   │   └── Services/          MovieDetailService.swift, MockMovieDetailService.swift,
│   │                          MovieCastService.swift, MockMovieCastService.swift,
│   │                          WatchlistMembershipService.swift, MockWatchlistMembershipService.swift
│   ├── MovieLive/
│   │   ├── Mapping/           Movie+TMDB.swift, CastMember+TMDB.swift, MovieError+TMDB.swift
│   │   └── Services/          TMDBMovieDetailService.swift, TMDBMovieCastService.swift
│   └── MovieUI/
│       ├── Models/            display mappings
│       ├── Navigation/        MovieDestination.swift
│       ├── Resources/         Localizable.xcstrings
│       ├── View Models/       MovieDetailViewModel.swift, MovieDetailViewModel+WatchlistMembership.swift,
│       │                      MovieCastViewModel.swift
│       └── Views/
│           ├── MovieDetail/   MovieDetailView.swift, MovieDetailStateView.swift, MovieDetailContentView.swift, …
│           ├── MovieCast/     MovieCastView.swift, MovieCastStateView.swift, MovieCastContentView.swift
│           └── Cards/         CastMemberCard.swift, which both screens show
└── Tests/MovieTests/
    ├── Live/                  MovieMappingTests.swift
    ├── ViewModels/            MovieDetailViewModelTests.swift, MovieCastViewModelTests.swift
    └── Fixtures.swift
```

One type per file, named after the type. `Views/` has a folder per screen, and one per kind of view several screens share.

## Package.swift template

```swift
// swift-tools-version: 6.4

import PackageDescription

let swiftSettings: [SwiftSetting] = [
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances"),
    .enableUpcomingFeature("MemberImportVisibility"),
]

let package = Package(
    name: "Discover",
    defaultLocalization: "en",
    platforms: [
        .iOS(.v27)
    ],
    products: [
        .library(name: "DiscoverCore", targets: ["DiscoverCore"]),
        .library(name: "DiscoverUI", targets: ["DiscoverUI"]),
        .library(name: "DiscoverLive", targets: ["DiscoverLive"]),
    ],
    dependencies: [
        .package(path: "../Shared"),
        .package(path: "../DesignSystem"),
        .package(path: "../Networking"),
    ],
    targets: [
        .target(
            name: "DiscoverCore",
            dependencies: [
                .product(name: "Shared", package: "Shared")
            ],
            swiftSettings: swiftSettings
        ),
        .target(
            name: "DiscoverUI",
            dependencies: [
                "DiscoverCore",
                .product(name: "Shared", package: "Shared"),
                .product(name: "SharedUI", package: "Shared"),
                .product(name: "DesignSystem", package: "DesignSystem"),
            ],
            resources: [
                .process("Resources")
            ],
            swiftSettings: swiftSettings
        ),
        .target(
            name: "DiscoverLive",
            dependencies: [
                "DiscoverCore",
                .product(name: "Shared", package: "Shared"),
                .product(name: "SharedLive", package: "Shared"),
                .product(name: "Networking", package: "Networking"),
            ],
            swiftSettings: swiftSettings
        ),
        .testTarget(
            name: "DiscoverTests",
            dependencies: [
                "DiscoverCore",
                "DiscoverUI",
                "DiscoverLive",
                .product(name: "Shared", package: "Shared"),
                .product(name: "SharedUI", package: "Shared"),
                .product(name: "Networking", package: "Networking"),
            ],
            swiftSettings: swiftSettings
        ),
    ],
    swiftLanguageModes: [.v6]
)
```

- Never add `.defaultIsolation(MainActor.self)` to a package. Domain models, errors and destinations stay nonisolated and their conformances aren't main-actor-isolated; UI types are marked `@MainActor` explicitly.
- No `@unchecked Sendable`, `nonisolated(unsafe)` or `@preconcurrency` in feature packages: a type that would need one becomes a value type or an actor.

## Shared, SharedUI and SharedLive

The shared package mirrors a feature's layers.

| Target | Contains | Imports |
|---|---|---|
| `Shared` | `Domain/`: the vocabulary several features and services share (`MovieSummary`, `Page<Item>`) | Foundation only |
| `SharedUI` | `Components/`: views several features show, those that render Shared types (`MovieSummaryCard`, `MovieSummaryRow`) and the shared screen states (`ContentUnavailableView.offline(retry:)`), with the copy they share. `Navigation/`: `SharedDestination`, the places several features open ([navigation](navigation.md#shareddestination-sharedui)) | `Shared`, `DesignSystem` |
| `SharedLive` | `Mapping/`: an API type's mapping to a Shared model, when several features read the same response as the same model (`MovieSummary+TMDB.swift`), tested in `SharedLiveTests` | `Shared`, `Networking` |

```swift
/// One page of a paginated list, as a service returns it.
public struct Page<Item: Sendable>: Sendable {
    public let items: [Item]
    /// The page to request next, or `nil` when this is the last one. Pages start at 1.
    public let nextPage: Int?

    public init(items: [Item], nextPage: Int?) {
        self.items = items
        self.nextPage = nextPage
    }
}
```

Promotion rule: move a type into Shared or SharedUI, or a mapping into SharedLive, only when a second feature needs the same concept and both copies must change together. The rule holds both ways: what Shared, SharedUI or SharedLive holds that only one feature uses belongs in that feature. If two types merely look alike today, duplicate them; they will diverge (a Discover poster card with a rating overlay stays in Discover even though a plain card lives in SharedUI).

Never in Shared or SharedUI:
- Models, views or mappings one feature uses, from however many of its screens (`Movie`, `CastMember`, `WatchlistItem`, `CastMemberCard`)
- Errors, service protocols or ports
- Navigation types other than `SharedDestination` (`XDestination` belongs to features, `Destination` and `Router` to the app)
- Environment keys, app state, preferences
- Generated API types or anything from infrastructure
- "Utils" grab-bags; a helper lives next to the concept it serves

Identifiers cross features as IDs (`MovieSummary.ID`), not as objects. Don't add a wrapper ID type unless the domain needs one. SharedLive holds only mappings: no services, models or errors, since each feature maps the client's errors to its own. There is no shared testing target: test fixtures are repeated in each feature that needs them.

## DesignSystem

One domain-free package, imported by every UI target, holding the visual language:
- **Tokens** under one namespace, by role: `DesignSystem.Colors.Text.primary`, `DesignSystem.Spacing.space5`. Tokens are values, never views.
- **Styles** for native controls, as conformances to SwiftUI's style protocols exposed as static members, so features keep using `Button`, `NavigationLink` and `Toggle` (`.buttonStyle(.content)`).
- **Components** that take plain values and view builders, such as a rail or a remote image, and extensions that restyle native views rather than wrappers around them.

It never knows a feature model, service, destination or screen, has no strings of its own, and decides no policy: an image session or cache comes from the app. A view that renders a domain type and is used by several features belongs in SharedUI, built from DesignSystem parts.

## Networking and other infrastructure

- `Networking` generates the API's types and client with Apple's swift-openapi-generator and wraps them in the app's client ([networking](networking.md)). Persistence or analytics packages follow the same rules.
- Infrastructure is imported by the Live targets (XLive and SharedLive), XTests and the app, never by XCore or XUI. Its types never appear in a feature's public API.

## Localization, assets and fonts

Each module owns the strings, images and colors it shows, and reaches them through symbols Xcode generates, so a missing key or asset is a build error, not a blank at runtime.

**Strings**
- Each XUI target has a String Catalog (`Resources/Localizable.xcstrings`) for everything its views show; the app has one for its own chrome (tabs, settings). SharedUI's catalog holds only copy every feature must say the same way, such as the shared screen states. XCore, XLive, Shared and DesignSystem show no text. A string two features show is duplicated in each catalog.
- Add the key to the catalog first (`discover.navigation.title`), then use its generated symbol: `Text(.discoverNavigationTitle)`, `Button(.discoverActionRetry, action: retry)`, `.navigationTitle(.discoverNavigationTitle)`; a key with a placeholder generates a function (`.searchNoResultsMessage(query)`). Symbols are internal to their module and carry its bundle. A literal in a package would need `bundle: #bundle`, or it silently looks in the app's bundle.
- A non-view type that carries user-facing text holds a `LocalizedStringResource`, never a `String`, so it resolves at display time in the right locale and bundle. Display mappings are XUI extensions on domain values (`DiscoverCategory.title`); models and view models hold no display strings.

**Images and colors**
- Each module keeps an asset catalog for what it shows: a feature's artwork in its XUI target, shared icons, illustrations and colors in DesignSystem, the app icon in the app.
- Use the generated symbols: `Image(.emptyWatchlist)`, `Color(.accentHighlight)`, `Label(.watchlistActionAdd, image: .bookmark)`. Never `Image("name")`.

**Fonts**
- Custom fonts ship in DesignSystem's resources. A package has no Info.plist for `UIAppFonts`, so DesignSystem registers them itself, once, before the first view draws.

**Build setup**
- The app sets `LOCALIZATION_PREFERS_STRING_CATALOGS`, `STRING_CATALOG_GENERATE_SYMBOLS`, `SWIFT_EMIT_LOC_STRINGS` and `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS`.
- Each package with resources declares `defaultLocalization` and processes its `Resources` folder.

## Testing

Each feature package builds, previews and tests without the app.

| Behavior | Tested in |
|---|---|
| View model state, paging, keyed resources and modes ([what to cover](view-models.md#testing-view-models)) | `XTests/ViewModels`, with private actor stubs |
| Mapping API responses to models, and transport errors to the feature's errors | `XTests/Live`, with hand-built generated types |
| Requests, decoding, error classification and cancellation | `NetworkingTests`, with a stub transport |
| `Destination` mappings and adapters | app tests, when the app has a test target |
| Visual states (loading, empty, error, right-to-left) | previews |

Shared test data lives in the test target as `make…` factories (`makeMovie(id:title:)`) in `Fixtures.swift`. Run a feature's scheme (`Discover-Package`) after every change to it, and build the app after changing anything public.
