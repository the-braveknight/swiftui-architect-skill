# Domain models and services

XCore holds a feature's domain: the models its screens show, the errors they can fail with, and a service protocol per screen. XLive implements those protocols over the API and maps everything it receives into the domain. Nothing above the Live targets knows the API exists.

## Contents
- Domain models
- Errors
- Service protocols
- Naming services and their methods
- Live services
- Mocks
- Cross-feature services: consumer-owned ports

## Domain models

A domain model describes what the app shows, not what the API sends. The API's shape (optional everything, snake_case fields, image paths, date strings, envelopes, pagination metadata) stops at the Live targets.

```swift
// What TMDB sends, generated in Networking: every field optional, a date as text, an image as a path.
// Components.Schemas.MovieDetails(id:title:releaseDate:voteAverage:posterPath:genres:)

// What the Movie feature shows, in MovieCore:
public struct Movie: Identifiable, Equatable, Sendable {
    public let id: Int
    public let title: String
    public let year: Int?
    public let rating: Double?
    public let posterURL: URL?
    public let genres: [String]
}
```

- **Value types, `Equatable` and `Sendable`.** They cross from live services to the main actor, and an `@Observable` view model only skips redrawing for an equal value when it can compare them. `Identifiable` when a list shows them or a screen links to them.
- **`Identifiable` names the id's type.** Store `public let id: Int`, or whatever the id really is, and refer to it as `Movie.ID`, with no `typealias ID`.
- **The whole thing is a plain noun; what a list shows of it adds `Summary`**: `Movie` and `MovieSummary`. A summary links to its whole thing by id. A model name never ends in `Detail`, which only screens use, or in a transport word (`DTO`, `Response`).
- **A model is complete for its role.** When reusing a model would leave fields empty or filled with placeholders, that role gets its own type. The whole thing leaves out what loads separately, such as a [paged list](view-models.md#a-thing-and-its-paged-list).
- **Each fact is stated once, so an illogical value can't be written.** When two properties could disagree, carry one inside the other and compute it, rather than checking in an initializer:

```swift
// Can disagree: nothing stops the id from naming one genre and the source another.
public let id: DiscoverShelf.ID
public let source: Source

// Can't: the source says where the shelf comes from (`.genre(28)`), and the id follows.
public let source: Source
public var id: Source { source }
```

## Errors

Every feature has its own error enum, cased by what its screens show, or one per screen when their failures differ: Discover's feed can't be "not found", but a genre opened by its id can (`DiscoverError` and `GenreDetailError`). Errors are never shared between features.

```swift
public enum MovieError: Error, Equatable {
    case offline
    case notFound
    case unavailable
}
```

- Cases are situations a screen tells apart, not transport details. `unavailable` is the catch-all for every failure that isn't offline or not found.
- An empty answer is never an error. Never add a case, or throw one, to avoid an empty state the design doesn't have.

## Service protocols

XCore declares one service protocol per screen, holding only what that screen loads, with one ready-made method per resource. The service maps, aggregates and de-duplicates, so a view model only assigns what it returns.

```swift
/// What a movie's details show: the movie and its cast.
public protocol MovieDetailService: Sendable {
    /// - Throws: `MovieError`. Cancellation propagates as `CancellationError`.
    func movie(id: MovieSummary.ID) async throws -> Movie

    /// - Throws: `MovieError`. Cancellation propagates as `CancellationError`.
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}
```

- Methods return domain values ready to show. A paged list returns `Page<Item>` from Shared; a screen that pages one thing's list has [a method for each](view-models.md#a-thing-and-its-paged-list). Never wrap a page in a feature type to carry more: a count the screen shows is a read of its own.
- Methods use untyped `throws` and name the error in the doc comment. Cancellation must reach the view model as `CancellationError`, which `throws(MovieError)` couldn't carry.
- Protocols refine `Sendable`, so `any MovieDetailService` crosses into the environment and a main-actor view model.
- A concern with a source of its own, such as searches kept on the device, is a separate service the view model takes beside the screen's (`RecentSearchesService`). A capability another feature owns is a [port](#cross-feature-services-consumer-owned-ports).

## Naming services and their methods

Names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/) and Apple's own async APIs. The networking client keeps the spec's operation names; these rules start at the domain.

| Kind | Rule | Examples |
|---|---|---|
| Service protocol | The role's plain noun; protocols that describe what something is read as nouns | `MovieDetailService`, `DiscoverService` |
| Implementation | Its source, then the protocol's name, like `JSONDecoder` for `Decoder` | `TMDBMovieDetailService`, `SwiftDataWatchlistService` |
| Mock | `Mock` + the protocol's name, exposed as `.mock` | `MockMovieDetailService` |
| Reads | Noun phrases for what they return, with no side effects, like `WeatherService.weather(for:)` | `movie(id:)`, `cast(movieID:)`, `shelves(page:)` |
| Changes | Imperative verb phrases | `addToWatchlist(_:)`, `removeFromWatchlist(movieID:)` |
| Booleans | Assertions | `isInWatchlist(movieID:)` |
| Labels | `id:` for the thing itself, `<owner>ID:` for what belongs to an owner, `page:` last | `movie(id:)`, `movies(genreID:page:)` |

The split keeps call sites readable: the view model's `loadShelves()` changes state and calls the service's `shelves(page:)`, which returns a value.

## Live services

XLive implements each protocol over the networking client. A live service is a struct holding the client, or an actor when it holds state or a non-`Sendable` store such as a SwiftData context.

```swift
public struct TMDBMovieDetailService: MovieDetailService {
    private let client: TMDBClient

    public init(client: TMDBClient) {
        self.client = client
    }

    @concurrent
    public func movie(id: MovieSummary.ID) async throws -> Movie {
        do {
            let response = try await client.movieDetails(id: id)
            guard let movie = Movie(response) else {
                throw MovieError.unavailable
            }
            return movie
        } catch let error as TMDBError {
            throw MovieError(error)
        }
    }

    // cast(movieID:) has the same shape.
}
```

- Methods are `@concurrent`, so requests and mapping run off the main actor. Protocol requirements and mocks stay unannotated.
- Mapping lives in `Model+Source.swift` extensions (`Movie+TMDB.swift`, `MovieError+TMDB.swift`). A model maps from its response in a failable initializer: a list item missing a required field is dropped, and a whole response that can't be read is `.unavailable`. A mapping to a Shared model that several features read lives once, in SharedLive ([modules](modules.md#shared-sharedui-and-sharedlive)).
- The feature's error maps from the client's error in an initializer, such as a 404 to `.notFound`. `CancellationError` passes through untouched ([networking](networking.md#errors-and-cancellation)).
- A step two live services share, such as reading extra artwork, is an internal extension on the client in XLive.

## Mocks

XCore ships a public mock of every protocol, so previews and other modules' tests need no network:

```swift
public struct MockMovieDetailService: MovieDetailService {
    /// What the mock answers: sample data, or a failure to preview an error state.
    public enum Scenario: Sendable {
        case sample
        case failure(MovieError)
    }

    private let scenario: Scenario

    public init(scenario: Scenario = .sample) {
        self.scenario = scenario
    }

    public func movie(id: MovieSummary.ID) async throws -> Movie {
        if case .failure(let error) = scenario { throw error }
        … sample data …
    }

    public func cast(movieID: MovieSummary.ID) async throws -> [CastMember] { … }
}

extension MovieDetailService where Self == MockMovieDetailService {
    public static var mock: Self { Self() }

    public static func mock(_ scenario: MockMovieDetailService.Scenario) -> Self { Self(scenario: scenario) }
}
```

- A scenario lets each preview show one state the screen really has; `.mock` stays the plain sample.
- A mock that keeps state, such as a watchlist that remembers toggles, is an actor, and `.mock` returns a fresh instance.
- Tests use private stubs in XTests instead ([testing](modules.md#testing)).

## Cross-feature services: consumer-owned ports

When feature A needs a capability that feature B owns, A declares a port in its own XCore, named for its own need. The app implements it with an adapter over B, so neither feature imports the other.

```swift
// MovieCore: named for the Movie feature's need, not after Watchlist.
public protocol WatchlistMembershipService: Sendable {
    func isInWatchlist(movieID: MovieSummary.ID) async throws -> Bool
    func addToWatchlist(_ movie: MovieSummary) async throws
    func removeFromWatchlist(movieID: MovieSummary.ID) async throws
}

// App/Adapters/WatchlistMembershipAdapter.swift
struct WatchlistMembershipAdapter: WatchlistMembershipService {
    let watchlist: any WatchlistService

    func addToWatchlist(_ movie: MovieSummary) async throws {
        try await watchlist.add(WatchlistItem(movie: movie, addedAt: .now))
    }
    …
}
```

The consumer ships a mock of its port too, so its previews and tests don't need the provider.
