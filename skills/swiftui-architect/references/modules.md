# Modules

## Contents
- Feature packages: what a feature is, its four targets
- Folder layout
- Package.swift template
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
│   │   └── Services/          MovieDetailServiceProtocol.swift, MockMovieDetailService.swift
│   ├── MovieDetailLive/
│   │   ├── Mapping/           Movie+TMDB.swift, MovieDetailError+TMDB.swift
│   │   └── Services/          MovieDetailService.swift
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

## Services, errors and DTO mapping

XCore defines the contract in domain terms. One ready-made method per resource a screen shows, kept as readable as the [Home example](feature-ui.md#several-resources-and-pagination); mapping, aggregation and de-duplication live in the service, so view models only assign what it returns.

```swift
public protocol MovieDetailServiceProtocol: Sendable {
    /// - Throws: `MovieDetailError`. Cancellation propagates as `CancellationError`.
    func movie(id: MovieSummary.ID) async throws -> Movie
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}
```

Every feature has its own error enum, cased by what its screens show. Never centralize errors into a shared type.

```swift
public enum MovieDetailError: Error, Equatable, Sendable {
    case offline
    case notFound
    case unavailable
}
```

XLive maps transport errors and DTOs into the feature's types. Mapping lives in `Model+Source.swift` extensions; a DTO missing a required field maps to `nil` and is dropped.

```swift
extension CastMember {
    init?(_ credit: Components.Schemas.CastCredit) {
        guard let creditID = credit.creditId, let personID = credit.id, let name = credit.name else {
            return nil
        }
        self.init(
            id: creditID,
            personID: personID,
            name: name,
            character: credit.character,
            profileURL: TMDBImage.url(path: credit.profilePath, size: .profile)
        )
    }
}

extension MovieDetailError {
    init(_ error: TMDBError) {
        switch error {
        case .offline: self = .offline
        case .unexpectedStatus(404, _): self = .notFound
        case .unexpectedStatus, .invalidResponse: self = .unavailable
        }
    }
}
```

The live service catches the transport error and rethrows the feature's: `catch let error as TMDBError { throw MovieDetailError(error) }`.

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
| `CoreUI` | `Components/`: views that render Core types and that several features show (`MovieSummaryCard`, `MovieSummaryRow`) | `Core`, `DesignSystem` |

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
