# Modules

## Contents
- Feature packages: what a feature is, its four targets
- Folder layout
- Package.swift template
- Models
- Services, errors and DTO mapping
- Cross-feature services: consumer-owned ports
- Core and CoreUI
- DesignSystem and infrastructure packages

## Feature packages

A feature is a vertical slice around one capability, and it owns that capability's data contract. It may export several screens (MovieDetail exports the movie screen and the cast screen); screens never define the boundary. Name the package after the capability: `Discover`, `Search`, `Watchlist`, `MovieDetail`, `PersonDetail`.

| Target | Contains | Imports |
|---|---|---|
| `XCore` | Models, `XError`, `XServiceProtocol`, `MockXService` | `Core` |
| `XUI` | View models, views, `XDestination`, `Localizable.xcstrings`, model → display mappings | `XCore`, `Core`, `CoreUI`, `DesignSystem` |
| `XLive` | `XService` (the live implementation), DTO → model mapping, persistence | `XCore`, `Core`, infrastructure |
| `XTests` | View model tests, mapping tests (Swift Testing) | all of the above, plus infrastructure for DTO fixtures |

XCore has no SwiftUI and no infrastructure, so models and contracts stay testable and reusable by any UI.

## Folder layout

```
Packages/MovieDetail/
├── Package.swift
├── Sources/
│   ├── MovieDetailCore/
│   │   ├── Models/            Movie.swift, CastMember.swift, MovieDetailError.swift
│   │   └── Services/          MovieDetailServiceProtocol.swift, MockMovieDetailService.swift,
│   │                          MovieCastServiceProtocol.swift, MockMovieCastService.swift
│   ├── MovieDetailLive/
│   │   ├── Mapping/           Movie+TMDB.swift, MovieDetailError+TMDB.swift
│   │   └── Services/          MovieDetailService.swift, MovieCastService.swift
│   └── MovieDetailUI/
│       ├── Navigation/        MovieDetailDestination.swift
│       ├── Resources/         Localizable.xcstrings
│       ├── View Models/       MovieDetailViewModel.swift, MovieCastViewModel.swift
│       └── Views/             MovieDetailView.swift, MovieDetailStateView.swift, MovieDetailContentView.swift, …
└── Tests/MovieDetailTests/
    ├── Live/                  MovieDetailMappingTests.swift
    └── ViewModels/            MovieDetailViewModelTests.swift
```

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

Never add `.defaultIsolation(MainActor.self)` to a package; mark UI types `@MainActor` explicitly.

## Models

XCore models describe what the screens show, not the API's responses.

- **The whole thing is a plain noun; what a list shows of it adds `Summary`**: `Movie` and `MovieSummary`, `HomeSection` and `HomeSectionSummary`. A summary links to its whole thing by id.
- **A model is complete for its role.** If reusing a model somewhere would leave fields empty or filled with placeholders, that role gets its own type. The whole thing also leaves out what loads separately, such as a [paged list](feature-ui.md#a-thing-and-its-paged-list).
- **Illogical values can't be written, because each fact is stated once.** When two properties could disagree, such as an id saying where a section comes from and content saying what it holds, carry one inside the other and compute it, rather than checking in an initializer. A value loaded by its id doesn't store the id again unless something reads it.

```swift
// Can disagree: nothing stops a genre's id from holding a list's content.
public let id: HomeSection.ID
public let content: Content

// Can't: the content says where it comes from (`.movies(from: .genre(28), items: …)`), and the id follows.
public let content: Content
public var id: HomeSection.ID { … }
```

## Services, errors and DTO mapping

XCore defines the contract in domain terms. Each screen has its own protocol, declaring only what that screen loads, with one ready-made method per resource, kept as readable as the [Home example](feature-ui.md#several-resources-and-pagination). Mapping, aggregation and de-duplication live in the service, so view models only assign what it returns.

```swift
public protocol MovieDetailServiceProtocol: Sendable {
    /// - Throws: `MovieDetailError`. Cancellation propagates as `CancellationError`.
    func movie(id: MovieSummary.ID) async throws -> Movie
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}

public protocol MovieCastServiceProtocol: Sendable {
    /// - Throws: `MovieDetailError`. Cancellation propagates as `CancellationError`.
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}
```

Each protocol has one live service in XLive and one mock in XCore. A step two live services share, such as reading extra artwork for their results, is an internal extension on the client in XLive, not a service conforming to two protocols.

Every feature has its own error enum, cased by what its screens show. Never centralize errors into a shared type.

```swift
public enum MovieDetailError: Error, Equatable, Sendable {
    case offline
    case notFound
    case unavailable
}
```

XLive maps transport errors and DTOs into the feature's types, in `Model+Source.swift` extensions (`CastMember+TMDB.swift`, `MovieDetailError+TMDB.swift`):
- A model maps from its DTO in a failable initializer; a DTO missing a required field maps to `nil` and is dropped.
- The feature's error maps from the transport error in an initializer, such as a 404 to `.notFound`. The live service catches the transport error and rethrows the feature's: `catch let error as TMDBError { throw MovieDetailError(error) }`.

## Cross-feature services: consumer-owned ports

When feature A needs a capability that feature B owns, A declares a port in its own vocabulary, named for its own need. The app implements the port with an adapter over B. Neither feature imports the other.

```swift
// MovieDetailCore: named for MovieDetail's need, not after Watchlist
public protocol WatchlistMembershipServiceProtocol: Sendable {
    func isInWatchlist(movieID: MovieSummary.ID) async throws -> Bool
    func addToWatchlist(_ movie: MovieSummary) async throws
    func removeFromWatchlist(movieID: MovieSummary.ID) async throws
}

// App/Adapters/WatchlistMembershipAdapter.swift
struct WatchlistMembershipAdapter: WatchlistMembershipServiceProtocol {
    let watchlist: WatchlistService

    func addToWatchlist(_ movie: MovieSummary) async throws {
        try await watchlist.add(WatchlistItem(movie: movie, addedAt: .now))
    }
    …
}
```

The consumer also ships a mock of its port in XCore (`MockWatchlistMembershipService`), so its previews and tests don't need the provider. Use the same pattern when a screen needs data another feature owns.

Escalation for a capability with many consumers: the provider publishes a small interface target (`XInterface`) with the protocol and its models. Use it only when ports start duplicating each other.

## Core and CoreUI

The shared package mirrors a feature's layers.

| Target | Contains | Imports |
|---|---|---|
| `Core` | `Domain/`: the vocabulary several features and services share (`MovieSummary`, `Page<Item>`) | Foundation only |
| `CoreUI` | `Components/`: views several features show, those that render Core types (`MovieSummaryCard`, `MovieSummaryRow`) and the shared screen states (`ContentUnavailableView.offline(retry:)`), with the copy they share | `Core`, `DesignSystem` |

Promotion rule: move a type into Core or CoreUI only when a second feature needs the same concept and both copies must change together. If two types merely look alike today, duplicate them; they will diverge (a Discover poster card with a rating overlay stays in Discover even though a plain card lives in CoreUI).

Never in Core or CoreUI:
- Models used by one feature (`Movie`, `Person`, `CastMember`, `WatchlistItem`)
- Errors, service protocols or ports
- Navigation types (`XDestination` belongs to features, `Destination` and `Router` to the app)
- Environment keys, app state, preferences
- DTOs or anything from infrastructure
- "Utils" grab-bags; a helper lives next to the concept it serves

Identifiers cross features as IDs (`MovieSummary.ID`), not as objects. Don't add a wrapper ID type unless the domain needs one.

There is no shared Live or testing target: mapping a transport type to a Core model (`MovieSummary+TMDB.swift`) and test fixtures are repeated in each feature that needs them.

## DesignSystem and infrastructure packages

- `DesignSystem` imports SwiftUI only and knows no feature.
- Infrastructure packages (`Networking`, persistence, analytics) are imported by XLive targets and the app, never by XCore or XUI. Their types never appear in a feature's public API.
- Policies such as an image `URLSession` or cache sizes belong to the app, which injects them (`.asyncImageURLSession(imageSession)`), never to DesignSystem or a feature.
