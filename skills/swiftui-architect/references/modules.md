# Modules

The app is built from local Swift packages: one per feature, plus a few shared packages every feature stands on. The compiler enforces the boundaries, since a module can use only what its package declares.

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

A feature is a vertical slice around one capability, and it owns that capability's data contract and every screen of it: Movie owns a movie's details and its cast. Screens and tabs never define the boundary. Name the package after the capability, never after a tab or a screen: `Discover`, `Movie`, `Search`, `Watchlist`, not `MovieDetail` or `Home`.

| Target | Contains | Imports |
|---|---|---|
| `XCore` | Domain models, `XError`, a service protocol and mock per screen, ports | `Shared` |
| `XUI` | View models, views, `XDestination`, `Localizable.xcstrings`, display mappings | `XCore`, `Shared`, `SharedUI`, `DesignSystem` |
| `XLive` | Live services, mapping API responses and errors to the domain, persistence | `XCore`, `Shared`, `SharedLive`, `Networking` |
| `XTests` | View model and mapping tests (Swift Testing) | all of the above, plus `Networking` for response fixtures |

- XCore has no SwiftUI and no infrastructure, so the domain stays testable and usable by any UI.
- XUI depends on XCore's protocols, never on XLive, so it builds, previews and tests on mocks alone.
- XLive is the only feature target that sees the API.
- The app links all three library targets: it builds live services from XLive and shows views from XUI ([composition](composition.md)).

## Inside a feature

Names inside a feature say what a thing is, so a screen, model or view keeps its name whichever screens or tabs show it.

| What | Named for | Example |
|---|---|---|
| A tab's root screen, with its view model, service, error and the models only it shows | Its tab | `DiscoverView`, `DiscoverViewModel`, `DiscoverService`, `DiscoverShelf` |
| Any other screen | What it shows | `GenreDetailView`, `MovieCastView` |
| A view, model or mapping several of the feature's screens use | What it is | `CastMemberCard`, `MovieSummaryGrid` |
| What the whole feature owns | The feature | `MovieDestination`, `MovieLayout`, the `movie.` string keys |

- A feature that several tabs show owns each tab's root screen. If Discover split into Movies and TV tabs, it would own `MoviesView` and `TVShowsView`, and the screens both open.
- A section two of the feature's screens show is one view in `Components/`, filed by kind (`Cards/`, `Rails/`): the movie's details and its cast screen both show `CastMemberCard`. A view that shows a model is named for the model and its form (`CastMemberCard`, `MovieSummaryGrid`), and keeps that name if it moves to SharedUI. A frame several screens put around different content takes that content as view builders.
- A name an imported module already declares takes a qualifier from the domain: `MovieCollection`, not `Collection` (Swift); `MovieLayout`, not `Layout` (SwiftUI). Unqualified, it hides the module's type everywhere in the feature.

## Folder layout

```
Packages/Movie/
├── Package.swift
├── Sources/
│   ├── MovieCore/
│   │   ├── Models/            Movie.swift, CastMember.swift, MovieError.swift
│   │   └── Services/
│   │       ├── MovieDetailService/          MovieDetailService.swift, MockMovieDetailService.swift
│   │       ├── MovieCastService/            MovieCastService.swift, MockMovieCastService.swift
│   │       └── WatchlistMembershipService/  WatchlistMembershipService.swift, MockWatchlistMembershipService.swift
│   ├── MovieLive/
│   │   ├── Mapping/           Movie+TMDB.swift, CastMember+TMDB.swift, MovieError+TMDB.swift
│   │   └── Services/          TMDBMovieDetailService.swift, TMDBMovieCastService.swift
│   └── MovieUI/
│       ├── MovieDetail/       MovieDetailViewModel.swift, MovieDetailViewModel+WatchlistMembership.swift,
│       │                      MovieDetailView.swift, MovieDetailStateView.swift, MovieDetailContentView.swift, …
│       ├── MovieCast/         MovieCastViewModel.swift, MovieCastView.swift, MovieCastStateView.swift,
│       │                      MovieCastContentView.swift
│       ├── Components/        MovieLayout.swift
│       │   └── Cards/         CastMemberCard.swift
│       ├── Navigation/        MovieDestination.swift
│       └── Resources/         Localizable.xcstrings
└── Tests/MovieTests/
    ├── Live/                  MovieMappingTests.swift
    ├── ViewModels/            MovieDetailViewModelTests.swift, MovieCastViewModelTests.swift
    └── Fixtures.swift
```

Where each file goes: [folders](conventions.md#folders).

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
    name: "Movie",
    defaultLocalization: "en",
    platforms: [
        .iOS(.v27)
    ],
    products: [
        .library(name: "MovieCore", targets: ["MovieCore"]),
        .library(name: "MovieUI", targets: ["MovieUI"]),
        .library(name: "MovieLive", targets: ["MovieLive"]),
    ],
    dependencies: [
        .package(path: "../Shared"),
        .package(path: "../DesignSystem"),
        .package(path: "../Networking"),
    ],
    targets: [
        .target(
            name: "MovieCore",
            dependencies: [
                .product(name: "Shared", package: "Shared")
            ],
            swiftSettings: swiftSettings
        ),
        .target(
            name: "MovieUI",
            dependencies: [
                "MovieCore",
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
            name: "MovieLive",
            dependencies: [
                "MovieCore",
                .product(name: "Shared", package: "Shared"),
                .product(name: "SharedLive", package: "Shared"),
                .product(name: "Networking", package: "Networking"),
            ],
            swiftSettings: swiftSettings
        ),
        .testTarget(
            name: "MovieTests",
            dependencies: [
                "MovieCore",
                "MovieUI",
                "MovieLive",
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

- Use the current tools version and platforms; the settings above are what the architecture relies on.
- Never add `.defaultIsolation(MainActor.self)` to a package. Generated OpenAPI code doesn't build under it, and domain models, errors and destinations must stay nonisolated; UI types are marked `@MainActor` explicitly. Only the app target may default to the main actor.
- No `@unchecked Sendable`, `nonisolated(unsafe)` or `@preconcurrency` in feature packages: a type that would need one becomes a value type or an actor.

## Shared, SharedUI and SharedLive

The shared package mirrors a feature's layers:

| Target | Contains | Imports |
|---|---|---|
| `Shared` | `Domain/`: the vocabulary several features share (`MovieSummary`, `Page<Item>`) | Foundation only |
| `SharedUI` | `Components/`: views several features show (`MovieSummaryCard`) and the shared screen states (`ContentUnavailableView.offline(retry:)`) with their copy. `Navigation/`: `SharedDestination` ([navigation](navigation.md#shareddestination-sharedui)) | `Shared`, `DesignSystem` |
| `SharedLive` | `Mapping/`: an API type's mapping to a Shared model that several features read (`MovieSummary+TMDB.swift`), tested in `SharedLiveTests` | `Shared`, `Networking` |

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

**The promotion rule.** A type moves into Shared or SharedUI, and a mapping into SharedLive, only when a second feature needs the same concept and both copies must change together. It holds both ways: whatever is shared but used by one feature, from however many of its screens, belongs in that feature. Two types that merely look alike are duplicated; they will diverge.

Never in the shared package:
- Errors, service protocols, ports, services or persistence; each feature maps the client's errors to its own
- Navigation types other than `SharedDestination`
- Environment keys, app state or preferences
- Generated API types outside SharedLive
- "Utils" grab-bags; a helper lives next to the concept it serves

Identifiers cross features as IDs (`MovieSummary.ID`), not as objects. Test fixtures are repeated in each feature that needs them, with no shared testing target.

## DesignSystem

One domain-free package, imported by every UI target, holding the visual language:
- **Tokens** under one namespace, by role: `DesignSystem.Colors.Text.primary`, `DesignSystem.Spacing.space5`.
- **Styles** for native controls, as conformances to SwiftUI's style protocols exposed as static members, so features keep using `Button` and `NavigationLink` (`.buttonStyle(.content)`).
- **Components** that take plain values and view builders, such as a rail or a remote image.

It never knows a feature model, service, destination or screen, has no strings of its own, and decides no policy: an image cache comes from the app. A view that renders a domain type belongs in a feature, or in SharedUI when several features show it.

## Networking and other infrastructure

`Networking` generates the API's types and client and wraps them in the app's client ([networking](networking.md)). It and any persistence or analytics package are imported by the Live targets, XTests and the app, never by XCore or XUI, and their types never appear in a feature's public API.

## Localization, assets and fonts

Each module owns the strings, images and colors it shows, and reaches them through symbols Xcode generates, so a missing key or asset is a build error, not a blank at runtime.

- **Strings.** Each XUI target has a String Catalog (`Resources/Localizable.xcstrings`) for everything its views show; the app has one for its own chrome. SharedUI's catalog holds only copy every feature must say the same way, such as the shared screen states. A string two features show is duplicated in each catalog.
- Add the key first (`movie.cast.title`), then use its generated symbol: `Text(.movieCastTitle)`, `Button(.movieActionRetry, action: retry)`; a key with a placeholder generates a function. Symbols carry their module's bundle. A string literal in a package would need `bundle: #bundle`, or it silently looks in the app's bundle.
- A non-view type that carries user-facing text holds a `LocalizedStringResource`, never a `String`. Display mappings are XUI extensions on domain values (`Department.title`); models and view models hold no display strings.
- **Images and colors.** A feature's artwork lives in its XUI target, shared icons and colors in DesignSystem, the app icon in the app. Use the generated symbols (`Image(.emptyWatchlist)`), never `Image("name")`.
- **Fonts** ship in DesignSystem, which registers them itself, once, since a package has no Info.plist.
- The app sets `LOCALIZATION_PREFERS_STRING_CATALOGS`, `STRING_CATALOG_GENERATE_SYMBOLS`, `SWIFT_EMIT_LOC_STRINGS` and `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS`; each package with resources declares `defaultLocalization`.

## Testing

Each feature package builds, previews and tests without the app. Test what a bug would break, not what every instance of a shape repeats.

| Behavior | Tested in |
|---|---|
| A screen's own rules: what it loads, its failure rule, paging, stale results ([which tests](view-models.md#testing-view-models)) | `XTests/ViewModels`, with private actor stubs |
| The API's quirks in mapping: dropped incomplete items, duplicates, paging metadata | `XTests/Live` and `SharedLiveTests`, with hand-built generated types |
| Requests, decoding, error classification and cancellation | `NetworkingTests`, with a stub transport |
| `Destination` mappings that aren't one-to-one, and adapters | app tests |
| Visual states: loading, empty, error, right-to-left | previews |

Test data comes from `make…` factories (`makeMovie(id:title:)`) in each test target's `Fixtures.swift`. Run the feature's scheme (`Movie-Package`) after every change to it, and build the app after changing anything public.
