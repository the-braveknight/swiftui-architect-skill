# Testing

Each feature package builds, previews and tests without the app. The package boundary is what makes that possible: XUI depends on XCore's protocol, never on XLive, so a mock or stub is all a view model needs.

## Contents
- Levels of isolation
- Mocks and stubs
- View model tests
- Live mapping tests
- What to test where
- Running tests

## Levels of isolation

| Level | Runs | Needs |
|---|---|---|
| Feature tests | `XTests` against XCore, XUI, XLive | the feature package and its dependencies only |
| Feature previews | Every public view, on `.mock` services | the feature package only |
| App | Composition: Destination mappings, adapters, screens | everything |

A feature that needs the app to be tested or previewed has a dependency pointing the wrong way.

## Mocks and stubs

- **Mock** (in XCore, public): a working sample implementation of the service protocol for previews and the app's sample mode, exposed as `.mock`.

```swift
public struct MockDiscoverService: DiscoverServiceProtocol {
    public init() {}

    public func loadShelves() async throws -> [DiscoverShelf] { … sample data … }
}

extension DiscoverServiceProtocol where Self == MockDiscoverService {
    public static var mock: Self { Self() }
}
```

- A mock that keeps state (a watchlist that remembers toggles) is an `actor`, and `.mock` returns a fresh instance.
- **Stub** (in XTests, private): a scripted implementation for one test file. Stubs that hold state are actors. An actor's synchronous method satisfies an `async` requirement and a non-throwing one satisfies `throws`, so a stub repeats `async`/`throws` only when it needs them.

```swift
/// Returns each queued result in turn, one per call.
private actor StubMovieDetailService: MovieDetailServiceProtocol {
    private var results: [Result<Movie, any Error>]

    init(_ results: Result<Movie, any Error>...) {
        self.results = results
    }

    func movie(id: MovieSummary.ID) throws -> Movie {
        try results.removeFirst().get()
    }

    func cast(movieID: MovieSummary.ID) -> [CastMember] {
        []
    }
}
```

## View model tests

Swift Testing, `@MainActor` suites, one behavior per test. Import the UI target with `@testable import XUI` to reach internal state and methods. Cover the loading rules the screen follows ([the proposed shape](feature-ui.md#loading-and-pagination-a-proposed-shape)): a resource loaded on appear starts out loading, the data loads, the error is recorded, a failed refresh keeps the data, a successful retry clears the error, and cancellation isn't an error. A paged screen also covers: a failed page keeps what's loaded, a retried page loads, loading the first page again clears the next page's error, a page that arrives after a refresh is dropped, and a restarted footer still loads its page.

```swift
@MainActor
struct MovieDetailViewModelTests {
    @Test func loadPublishesTheMovie() async {
        let movie = makeMovie(id: 7)
        let viewModel = MovieDetailViewModel(movieID: 7, service: StubMovieDetailService(.success(movie)))

        await viewModel.load()

        #expect(viewModel.movie == movie)
    }

    @Test func failedRefreshKeepsTheMovie() async {
        let movie = makeMovie(id: 7)
        let viewModel = MovieDetailViewModel(
            movieID: 7,
            service: StubMovieDetailService(.success(movie), .failure(MovieDetailError.offline))
        )

        await viewModel.load()
        await viewModel.load()

        #expect(viewModel.movie == movie)
        #expect(viewModel.movieError == .offline)
    }
}
```

Shared test data lives in the test target as `make…` factories (`makeMovie(id:title:)`, `makePerson(id:)`) in `Fixtures.swift`.

## Live mapping tests

Test the DTO → model mapping and the transport error → feature error mapping in XLive, without the network:

```swift
@testable import DiscoverLive

struct DiscoverMappingTests {
    @Test func mapsTransportErrors() {
        #expect(DiscoverError(TMDBError.offline) == .offline)
        #expect(DiscoverError(TMDBError.invalidResponse) == .unavailable)
    }

    @Test func dropsItemsWithoutAnIDOrTitle() {
        #expect(MovieSummary(Components.Schemas.MovieListItem(title: "No ID")) == nil)
        #expect(MovieSummary(Components.Schemas.MovieListItem(id: 1)) == nil)
    }
}
```

Infrastructure packages test themselves with a stub transport (requests built, responses decoded, errors classified).

## What to test where

| Behavior | Test in |
|---|---|
| View model state transitions | `XTests/ViewModels` |
| Focused models (`WatchlistMembership`) | `XTests/ViewModels` |
| DTO and error mapping, service assembly | `XTests/Live` |
| Paging (`loadNext…Page()`) and screen-wide `isLoading`/`error` | `XTests/ViewModels` |
| Requests, decoding, transport errors | the infrastructure package's tests |
| `Destination.init` mappings, adapters | app tests, when the app has a test target |
| Visual states (loading, empty, error, Arabic) | previews |

## Running tests

Each package has its own scheme (`Discover-Package`, or the product name for a single-product package):

```bash
cd Packages/Discover
xcodebuild test -scheme Discover-Package -destination 'platform=iOS Simulator,name=<an installed iPhone simulator>'
```

Run the feature's tests after every change to it, and build the app after changing anything public.
