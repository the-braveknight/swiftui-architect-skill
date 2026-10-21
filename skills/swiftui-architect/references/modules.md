# Modules

The app is built from local Swift packages: one per feature, plus a few shared packages every feature stands on. The compiler enforces the boundaries: a module can use only what its package declares.

## Contents
- Feature packages
- Folder layout
- Package.swift template
- Core and CoreUI
- DesignSystem
- Networking and other infrastructure
- Localization and assets
- Testing

## Feature packages

A feature is a vertical slice around one capability, and it owns that capability's data contract. It may export several screens (MovieDetail exports the movie screen and the cast screen); screens never define the boundary. Name the package after the capability: `Discover`, `Search`, `Watchlist`, `MovieDetail`, `PersonDetail`.

| Target | Contains | Imports |
|---|---|---|
| `XCore` | Domain models, `XError`, a service protocol and mock per screen, ports | `Core` |
| `XUI` | View models, views, `XDestination`, `Localizable.xcstrings`, display mappings | `XCore`, `Core`, `CoreUI`, `DesignSystem` |
| `XLive` | Live services, mapping API responses and errors to the domain, persistence | `XCore`, `Core`, `Networking` |
| `XTests` | View model tests and mapping tests (Swift Testing) | all of the above, plus `Networking` for response fixtures |

- XCore has no SwiftUI and no infrastructure, so the domain stays testable and usable by any UI ([domain](domain.md)).
- XUI depends on XCore's protocols, never on XLive, so it builds, previews and tests on mocks alone.
- XLive is the only feature target that sees the API.
- The app links all three library targets: it builds live services from XLive and shows views from XUI ([composition](composition.md)).

## Folder layout

```
Packages/MovieDetail/
├── Package.swift
├── Sources/
│   ├── MovieDetailCore/
│   │   ├── Models/            Movie.swift, CastMember.swift, MovieDetailError.swift
│   │   └── Services/          MovieDetailService.swift, MockMovieDetailService.swift,
│   │                          MovieCastService.swift, MockMovieCastService.swift,
│   │                          WatchlistMembershipService.swift, MockWatchlistMembershipService.swift
│   ├── MovieDetailLive/
│   │   ├── Mapping/           Movie+TMDB.swift, CastMember+TMDB.swift, MovieDetailError+TMDB.swift
│   │   └── Services/          TMDBMovieDetailService.swift, TMDBMovieCastService.swift
│   └── MovieDetailUI/
│       ├── Models/            display mappings
│       ├── Navigation/        MovieDetailDestination.swift
│       ├── Resources/         Localizable.xcstrings
│       ├── View Models/       MovieDetailViewModel.swift, MovieCastViewModel.swift, WatchlistMembership.swift
│       └── Views/             MovieDetailView.swift, MovieDetailStateView.swift, MovieDetailContentView.swift, …
└── Tests/MovieDetailTests/
    ├── Live/                  MovieDetailMappingTests.swift
    ├── ViewModels/            MovieDetailViewModelTests.swift, MovieCastViewModelTests.swift
    └── Fixtures.swift
```

One type per file, named after the type.

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
        .package(path: "../Core"),
        .package(path: "../DesignSystem"),
        .package(path: "../Networking"),
    ],
    targets: [
        .target(
            name: "DiscoverCore",
            dependencies: [
                .product(name: "Core", package: "Core")
            ],
            swiftSettings: swiftSettings
        ),
        .target(
            name: "DiscoverUI",
            dependencies: [
                "DiscoverCore",
                .product(name: "Core", package: "Core"),
                .product(name: "CoreUI", package: "Core"),
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
                .product(name: "Core", package: "Core"),
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
                .product(name: "Core", package: "Core"),
                .product(name: "CoreUI", package: "Core"),
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

## Core and CoreUI

The shared package mirrors a feature's layers.

| Target | Contains | Imports |
|---|---|---|
| `Core` | `Domain/`: the vocabulary several features and services share (`MovieSummary`, `Page<Item>`) | Foundation only |
| `CoreUI` | `Components/`: views several features show, those that render Core types (`MovieSummaryCard`, `MovieSummaryRow`) and the shared screen states (`ContentUnavailableView.offline(retry:)`), with the copy they share | `Core`, `DesignSystem` |

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

Promotion rule: move a type into Core or CoreUI only when a second feature needs the same concept and both copies must change together. If two types merely look alike today, duplicate them; they will diverge (a Discover poster card with a rating overlay stays in Discover even though a plain card lives in CoreUI).

Never in Core or CoreUI:
- Models used by one feature (`Movie`, `Person`, `CastMember`, `WatchlistItem`)
- Errors, service protocols or ports
- Navigation types (`XDestination` belongs to features, `Destination` and `Router` to the app)
- Environment keys, app state, preferences
- Generated API types or anything from infrastructure
- "Utils" grab-bags; a helper lives next to the concept it serves

Identifiers cross features as IDs (`MovieSummary.ID`), not as objects. Don't add a wrapper ID type unless the domain needs one. There is no shared Live or testing target: mapping an API type to a Core model (`MovieSummary+TMDB.swift`) and test fixtures are repeated in each feature that needs them.

## DesignSystem

One domain-free package, imported by every UI target, holding the visual language:
- **Tokens** under one namespace, by role: `DesignSystem.Colors.Text.primary`, `DesignSystem.Spacing.space5`. Tokens are values, never views.
- **Styles** for native controls, as conformances to SwiftUI's style protocols exposed as static members, so features keep using `Button`, `NavigationLink` and `Toggle`: `.buttonStyle(.chip)`, `.buttonStyle(.content)`. A variant is a parameter on one style (`.chip(.text)`), not a new type.
- **Components** that take plain values and view builders (`Rail(_:content:accessory:)`, `RemoteImage(url:)`), and initializers that restyle native views by extending them rather than wrapping them, such as an empty-state initializer on `ContentUnavailableView`.

It never knows a feature model, service, destination or screen, has no strings of its own, and decides no policy: an image `URLSession` or cache is injected by the app (`.asyncImageURLSession(imageSession)`). A view that renders a domain type and is used by several features belongs in CoreUI, built from DesignSystem parts.

## Networking and other infrastructure

- `Networking` generates the API's types and client with Apple's swift-openapi-generator and wraps them in the app's client ([networking](networking.md)). Persistence or analytics packages follow the same rules.
- Infrastructure is imported by XLive targets, XTests and the app, never by XCore or XUI. Its types never appear in a feature's public API.

## Localization and assets

- Each XUI target owns a String Catalog (`Resources/Localizable.xcstrings`) for everything its views show; the app has one for its own chrome (tabs, settings). CoreUI's catalog holds only copy every feature must say the same way, such as the shared screen states.
- XCore, XLive, Core and DesignSystem show no text. Content from the backend is already localized: the networking layer sends the app's language with each request.
- Views use the symbols Xcode generates (`Text(.discoverNavigationTitle)`, `Image(.emptyWatchlist)`), which are internal to their module and carry its bundle. A string two features show is duplicated in each catalog.
- Display strings never live in models or view models. An XUI extension maps a domain value to a `LocalizedStringResource` (`DiscoverCategory.title`).
- The app turns on `STRING_CATALOG_GENERATE_SYMBOLS` and `ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS`; each package with strings declares `defaultLocalization` and processes `Resources`.

## Testing

Each feature package builds, previews and tests without the app.

| Behavior | Tested in |
|---|---|
| View model state, paging and screen-wide `isLoading`/`error` ([what to cover](view-models.md#testing-view-models)) | `XTests/ViewModels`, with private actor stubs |
| Mapping API responses to models, and transport errors to the feature's errors | `XTests/Live`, with hand-built generated types |
| Requests, decoding, error classification and cancellation | `NetworkingTests`, with a stub transport |
| `Destination` mappings and adapters | app tests, when the app has a test target |
| Visual states (loading, empty, error, right-to-left) | previews |

Shared test data lives in the test target as `make…` factories (`makeMovie(id:title:)`) in `Fixtures.swift`. Run a feature's scheme (`Discover-Package`) after every change to it, and build the app after changing anything public.
