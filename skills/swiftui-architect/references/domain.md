# Domain models and services

XCore holds a feature's domain: the models its screens show, the errors they can fail with, and a service protocol per screen. XLive implements those protocols over the API and maps everything it receives into the domain. Nothing above XLive knows the API exists.

## Contents
- Domain models
- Errors
- Service protocols
- Naming services and their methods
- Live services
- Mocks
- Cross-feature services: consumer-owned ports

## Domain models

A domain model describes what the app wants to show, not what the API sends. The API's shape (optional everything, snake_case fields, image paths, date strings, envelopes, pagination metadata) stops at the Live targets, XLive and SharedLive.

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

- **Value types, `Equatable` and `Sendable`.** They cross from live services to the main actor, and an `@Observable` view model skips invalidating its views when a refresh assigns an equal value, which it can only compare when the type is `Equatable`. `Identifiable` when a list shows them or a screen links to them.
- **`Identifiable` provides the id's type.** Store `public let id: Int`, or whatever the id really is, and refer to it as `Movie.ID`; don't add `typealias ID`. Only a type that doesn't store its id, such as a section loaded by it, names the id's type with `typealias ID`.
- **The whole thing is a plain noun; what a list shows of it adds `Summary`**: `Movie` and `MovieSummary`, `HomeSection` and `HomeSectionSummary`. A summary links to its whole thing by id. A model name never ends in `Detail` (only screens do) or in a transport word (`DTO`, `Response`).
- **A model is complete for its role.** If reusing a model somewhere would leave fields empty or filled with placeholders, that role gets its own type. The whole thing also leaves out what loads separately, such as a [paged list](view-models.md#a-thing-and-its-paged-list).
- **Illogical values can't be written, because each fact is stated once.** When two properties could disagree, such as an id saying where a section comes from and content saying what it holds, carry one inside the other and compute it, rather than checking in an initializer. A value loaded by its id doesn't store the id again unless something reads it.

```swift
// Can disagree: nothing stops a genre's id from holding a list's content.
public let id: HomeSection.ID
public let content: Content

// Can't: the content says where it comes from (`.movies(from: .genre(28), items: …)`), and the id follows.
public let content: Content
public var id: HomeSection.ID { … }
```

A model used by one feature lives in its XCore; one that several features share, and that must change for all of them together, lives in Shared ([modules](modules.md#shared-sharedui-and-sharedlive)).

## Errors

Every feature has its own error enum, cased by what its screens show, or one per screen when their failures differ: Home's feed can't be "not found", but a genre opened by its id can (`HomeError` and `GenreDetailError`). Never centralize errors into a shared type.

```swift
public enum MovieError: Error, Equatable {
    case offline
    case notFound
    case unavailable
}
```

- Cases are situations a screen tells apart, not transport details: offline, what was opened doesn't exist, anything else. `unavailable` is the real catch-all for failures that aren't offline (timeouts, server errors, unreadable responses).
- An empty answer is never an error. Don't add a case, or throw one, to avoid an empty state the design doesn't have.
- Each error property in a view model has its own type. It's often the feature's `XError`, but a paging footer or one section can use a smaller enum when its view shows fewer cases.

## Service protocols

XCore declares one service protocol per screen, holding only what that screen loads, with one ready-made method per resource. The service does the mapping, aggregation and de-duplication, so a view model only assigns what it returns.

```swift
/// What the movie screen shows: the movie and its cast.
public protocol MovieDetailService: Sendable {
    /// - Throws: `MovieError`. Cancellation propagates as `CancellationError`.
    func movie(id: MovieSummary.ID) async throws -> Movie

    /// - Throws: `MovieError`. Cancellation propagates as `CancellationError`.
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}

/// What the cast screen shows.
public protocol MovieCastService: Sendable {
    /// - Throws: `MovieError`. Cancellation propagates as `CancellationError`.
    func cast(movieID: MovieSummary.ID) async throws -> [CastMember]
}
```

- Methods return domain values ready to show. A paged list returns `Page<Item>` from Shared (`items`, `nextPage`); a screen that pages one thing's list has two methods, [the thing and its list](view-models.md#a-thing-and-its-paged-list). Don't wrap a page in a feature type to carry more, such as a total: a count the screen shows is a read of its own (`resultCount(for:)`).
- Methods use untyped `throws` and name the error in the doc comment. Cancellation must reach the view model as `CancellationError`, and `throws(MovieError)` couldn't carry it without a cancellation case in every domain error.
- Protocols refine `Sendable`, so `any MovieDetailService` can be injected through the environment and held by a main-actor view model.
- A screen's service holds what the screen loads from one source. A concern with a source of its own, such as searches kept on the device in user defaults, is a separate service (`RecentSearchesService`, implemented by `UserDefaultsRecentSearchesService`) that the view model takes beside it. A [nested model](view-models.md#nested-models-and-modes) uses its view model's services and never gets one for being nested; a capability another feature owns is a [port](#cross-feature-services-consumer-owned-ports).

## Naming services and their methods

Names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/) and the shape of Apple's own async APIs. The networking client keeps the spec's operation names; these rules start at the domain.

| Kind | Rule | Examples |
|---|---|---|
| Service protocol | The role's plain noun. "Protocols that describe what something is should read as nouns"; the standard library adds `Protocol` only when a concrete type already owns the name (`StringProtocol`) | `MovieDetailService`, `HomeService` |
| Implementation | Named for where its data comes from, then the protocol's name, like `JSONDecoder` for `Decoder` | `TMDBMovieDetailService`, `SwiftDataWatchlistService` |
| Mock | `Mock` + the protocol's name, exposed as `.mock` | `MockMovieDetailService` |
| Reads | Noun phrases named for what they return: they have no side effects, like `WeatherService.weather(for:)` and `Product.products(for:)` | `movie(id:)`, `cast(movieID:)`, `sections(page:)`, `banners()` |
| Changes | Imperative verb phrases: they have side effects | `addToWatchlist(_:)`, `removeFromWatchlist(movieID:)` |
| Booleans | Read as assertions | `isInWatchlist(movieID:)` |
| Labels | `id:` for the thing itself, `<owner>ID:` for what belongs to an owner, `page:` last. IDs are weakly typed `Int`s, so the label states their role | `movie(id:)`, `movies(sectionID:page:)` |

The split keeps call sites readable: the view model's `loadSections()` changes state and calls the service's `sections(page:)`, which returns a value.

## Live services

XLive implements each protocol over the networking client. A live service is a struct holding the client; one that holds state or a non-`Sendable` store (a SwiftData context, a cache) is an actor.

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

- Methods are `@concurrent`, so requests and mapping run off the main actor of the view model that awaits them. Protocol requirements and mocks stay unannotated. For isolation and `Sendable` beyond this, use a Swift Concurrency skill ([related skills](../SKILL.md#related-skills)).
- Mapping lives in `Model+Source.swift` extensions (`Movie+TMDB.swift`, `MovieError+TMDB.swift`). A model maps from its response in a failable initializer: a list item missing a required field is dropped, and a whole response that can't be read is `.unavailable`. A mapping to a Shared model that several features read, such as `MovieSummary+TMDB.swift`, lives once, in SharedLive.
- The feature's error maps from the client's transport error in an initializer, such as a 404 to `.notFound`. `CancellationError` passes through untouched, because the client already rethrows a cancelled request as one ([networking](networking.md#errors-and-cancellation)).
- A step that two live services share, such as reading extra artwork for their results, is an internal extension on the client in XLive, not one service conforming to two protocols.

## Mocks

XCore ships a public mock of every protocol, so previews, the app's sample mode and other modules' tests need no network:

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

- A scenario lets each preview show one state the screen really has: a kind of sample (a movie or a series) or one of the feature's errors. `.mock` stays the plain sample.
- A mock that keeps state (a watchlist that remembers toggles) is an actor, and `.mock` returns a fresh instance.
- An actor's synchronous method satisfies an `async` requirement and a non-throwing one satisfies `throws`, so actor mocks and test stubs repeat `async` or `throws` only when they need them.
- Tests use private stubs in XTests instead ([testing](modules.md#testing)).

## Cross-feature services: consumer-owned ports

When feature A needs a capability that feature B owns, A declares a port in its own XCore, named for its own need. The app implements the port with an adapter over B. Neither feature imports the other.

```swift
// MovieCore: named for the Movie feature's need, not after Watchlist
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

The consumer ships a mock of its port too (`MockWatchlistMembershipService`), so its previews and tests don't need the provider. Escalation for a capability with many consumers: the provider publishes a small interface target (`XInterface`) with the protocol and its models. Use it only when ports start duplicating each other.
