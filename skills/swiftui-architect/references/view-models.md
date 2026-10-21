# View models

A view model loads what one screen shows and exposes it as state. Services do the work (mapping, aggregation, de-duplication) and return view-ready domain values; the view model assigns them and manages loading, errors and paging.

## Contents
- The shape
- One resource: data, activity and error
- Errors and cancellation
- Several resources
- Pagination
- A thing and its paged list
- Keyed resources
- Focused models and modes
- Testing view models

## The shape

```swift
@MainActor
@Observable
public final class DiscoverViewModel {
    private let service: any DiscoverService

    /// Shelves
    private(set) var shelves: [DiscoverShelf] = []
    private(set) var isLoadingShelves: Bool = true
    private(set) var shelvesError: DiscoverError?

    public init(service: any DiscoverService) {
        self.service = service
    }

    func load() async {
        isLoadingShelves = true
        defer { isLoadingShelves = false }

        do {
            let shelves = try await service.shelves()
            self.shelves = shelves
            self.shelvesError = nil
        } catch is CancellationError {

        } catch let error as DiscoverError {
            self.shelvesError = error
        } catch {
            self.shelvesError = .unavailable
        }
    }
}
```

- `@MainActor @Observable public final class`, one per screen and one per [mode](#modes), with a `public init` taking IDs and services. The app's `XScreen` builds one on every `body` pass and the view keeps only the first, so `init` does assignments only: no tasks, subscriptions or I/O.
- State is `private(set)`; actions are internal methods the views call (`load()`, `loadNextSectionsPage()`, `toggle(_:)`), named as imperative verbs because they change state. The one exception is input a view binds, such as a search field's text, which is a plain `var` ([keyed resources](#keyed-resources)).
- No data processing of its own: every value it stores comes from the service as it is.
- No navigation, no views, no display strings, no other feature. State holds domain values; display mappings are XUI extensions.
- `load()` is safe to call again (refresh, retry), with no `loadIfNeeded()` variants or "already loaded" latches.

## One resource: data, activity and error

This is a proposed shape for keeping loading, errors and paging simple, not a rule. Adapt it when a screen needs more, such as a refresh indicator or a banner for a failed refresh.

A screen can have data while it refreshes, data and a failed refresh, or an error while it retries. A single `idle / loading / loaded / failed` enum holds only one of these at a time, so each view model ends up rebuilding the others. Three plain properties per resource hold them all, with no shared state type:

| Property | Holds |
|---|---|
| The data, named for what it is (`shelves`, `movie`) | An empty collection, or `nil` for a single value, until the first successful load; a failed refresh keeps it |
| `isLoading…` (`isLoadingShelves`) | Whether its load is running. A resource the screen loads as soon as it appears starts `true`, so the first frame shows progress rather than the empty state |
| `…Error` (`shelvesError`) | An error the views switch over (`DiscoverError?`) |

Rules for each load method:
- Set `isLoading…` for the duration of the load (`defer`).
- On success, assign what the service returns and clear the error. On failure, keep the data and record the error, so a retry keeps the error view on screen until a load succeeds.
- State views read the data first: the content whenever there is data, then the error, progress and the empty state ([views](views.md#xview-xstateview-and-xcontentview)).

## Errors and cancellation

**Errors are the feature's own types.** Each error property has its own type, often the feature's `XError` ([domain](domain.md#errors)). Catch in this order:
1. `CancellationError`: nothing happened. Keep the data and the error; only the `defer` ends `isLoading…`.
2. `let error as XError`: record it.
3. Anything else: the fallback case of that property's type (`.unavailable`).

How tasks, isolation and cancellation work is a Swift Concurrency skill's subject ([related skills](../SKILL.md#related-skills)); what follows is how this shape relies on them.

**Cancellation is never a failure.** The networking client rethrows a cancelled request as `CancellationError` ([networking](networking.md#errors-and-cancellation)), so a view model needs only the one catch.
- `XView` runs `load()` from `.task`, which cancels it when the view disappears and runs it again when the view reappears (a tab switch, a pop back). The second run is a silent refresh: the data stays on screen.
- Start `load()` from `.task` on `XView`, never on the progress view. The progress view goes away when the first resource arrives, and cancelling its task would cancel the loads that follow.
- A retry button starts `Task { await viewModel.load() }`. Keep its main-actor start, since `load()` is main-actor-isolated and begins by setting state; never mark it `@concurrent`. The task isn't cancelled with the view, which is harmless because `load()` can run again.
- Loads may overlap, such as a retry during a refresh. Both ask for the same data, the last to finish wins, and `isLoading…` can end early. That's harmless too.

## Several resources

When a screen loads more than one resource, each gets its own properties and load method. The state view follows the resource the content can't do without; the others show when they arrive and are left out when they fail. Home loads its banners and a paged list of sections, and follows the sections:

```swift
public protocol HomeService: Sendable {
    /// - Throws: `HomeError`. Cancellation propagates as `CancellationError`.
    func banners() async throws -> [MovieSummary]

    /// - Throws: `HomeError`. Cancellation propagates as `CancellationError`.
    func sections(page: Int) async throws -> Page<HomeSectionSummary>
}

/// Home's banners and sections, with the sections paged in as the last one comes into view.
@MainActor
@Observable
public final class HomeViewModel {
    private let service: any HomeService

    /// Banners
    private(set) var banners: [MovieSummary] = []
    private(set) var isLoadingBanners: Bool = true
    private(set) var bannersError: HomeError?

    /// Sections
    private(set) var sections: [HomeSectionSummary] = []
    private(set) var isLoadingSections: Bool = true
    private(set) var sectionsError: HomeError?
    private(set) var nextSectionsPage: Int?
    private(set) var isLoadingNextSectionsPage = false
    private(set) var nextSectionsPageError: HomeError?

    public init(service: any HomeService) {
        self.service = service
    }

    func load() async {
        await withDiscardingTaskGroup { group in
            group.addTask {
                await self.loadBanners()
            }

            group.addTask {
                await self.loadSections()
            }
        }
    }

    // loadBanners() and loadSections() follow the one-resource shape; loadSections() asks for page 1.

    func loadNextSectionsPage() async { … }   // below
}
```

- Home's state view checks `sections`, `sectionsError` and `isLoadingSections`, so banners failing on their own don't fail Home.
- When a rule spans several resources, such as a screen that can't show anything until two of them load, the screen gets computed `isLoading` and `error` for its state view, with the type that view switches over. Never store them, since two loads would end up writing one value.
- `load()` runs the independent loads side by side in a discarding task group, which suits loads that return nothing. When a resource depends on another, such as episodes picked from the loaded title, `load()` loads that one first and then the rest side by side.

## Pagination

A paged resource adds three properties to its data, `isLoading…` and `…Error`:

| Property | Holds |
|---|---|
| `next…Page` (`nextSectionsPage`) | The page to ask for next, from the service's `Page<Item>`; `nil` after the last one |
| `isLoadingNext…Page` | Whether the next page is loading |
| `next…PageError` | Why the next page failed. Its own type: a footer often shows fewer cases than the error view |

```swift
func loadNextSectionsPage() async {
    guard let requestedPage = nextSectionsPage else {
        return
    }

    isLoadingNextSectionsPage = true
    defer { isLoadingNextSectionsPage = false }

    do {
        let page = try await service.sections(page: requestedPage)
        // A refresh may have started the sections over while this page loaded.
        guard nextSectionsPage == requestedPage else {
            return
        }
        self.sections.append(contentsOf: page.items)
        self.nextSectionsPage = page.nextPage
        self.nextSectionsPageError = nil
    } catch is CancellationError {

    } catch let error as HomeError {
        self.nextSectionsPageError = error
    } catch {
        self.nextSectionsPageError = .unavailable
    }
}
```

- Loading the first page again replaces the list, sets `next…Page` and clears `next…PageError`.
- `loadNext…Page()` remembers the page it asked for and, after the `await`, drops the result if the list no longer needs that page next: a refresh started it over, or a duplicate call already appended it.
- It has no in-flight guard. The footer's task restarts as the footer scrolls out and back in, and the restarted call must be able to load.
- Later pages append. No de-duplication across pages and no merging helpers on models; the service decides what a page holds.

## A thing and its paged list

A screen that shows one thing and a paged list belonging to it, such as a genre's "See all", loads them as two resources: the thing, and its list a page at a time. The screen has its own service:

```swift
/// A genre's "See all": the genre, then its movies a page at a time.
public protocol GenreDetailService: Sendable {
    /// The genre: its name, without its movies.
    func genre(id: Genre.ID) async throws -> Genre
    func movies(genreID: Genre.ID, page: Int) async throws -> Page<MovieSummary>
}

@MainActor
@Observable
public final class GenreDetailViewModel {
    private let genreID: Genre.ID
    private let service: any GenreDetailService

    /// Genre
    private(set) var genre: Genre?
    private(set) var isLoadingGenre: Bool = true
    private(set) var genreError: GenreDetailError?

    /// Movies
    private(set) var movies: [MovieSummary] = []
    private(set) var isLoadingMovies: Bool = true
    private(set) var moviesError: GenreDetailError?
    private(set) var nextMoviesPage: Int?
    private(set) var isLoadingNextMoviesPage = false
    private(set) var nextMoviesPageError: GenreDetailError?

    func load() async {
        // The genre doesn't decide whether there are movies, so both load side by side.
        await withDiscardingTaskGroup { group in
            group.addTask {
                await self.loadGenre()
            }

            group.addTask {
                await self.loadMovies()
            }
        }
    }

    // loadGenre(), loadMovies() and loadNextMoviesPage() follow the shapes above.
}
```

- The thing leaves the list out, so nothing holds two copies of it. Don't page a value that contains the list, merge pages into it, or pair it with its next page in a wrapper type or tuple.
- When the API returns the thing with its first page, this costs an extra request. Accept it for two plain resources.
- Load the list after the thing only when the thing decides whether there is one; otherwise load both side by side.
- The state view follows the list. The thing titles the screen from `XView`, so the title stays up in every state ([views](views.md#xview-xstateview-and-xcontentview)).

## Keyed resources

Some resources depend on a key the user changes on the screen: the words in a search, a selected season, a filter. The key is view model state, and the resource starts over whenever it changes.

```swift
/// Selects a season. Its episodes start over, and load from `loadEpisodes()`.
func select(_ season: Season) {
    guard season != selectedSeason else {
        return
    }

    selectedSeason = season
    episodes = []
    episodesError = nil
    // The view loads them next, so until then they show as loading rather than as none.
    isLoadingEpisodes = true
}

func loadEpisodes() async {
    guard let season = selectedSeason else {
        return
    }

    isLoadingEpisodes = true
    // A load for a season no longer selected leaves the flag to the selected season's load.
    defer {
        if season == selectedSeason {
            isLoadingEpisodes = false
        }
    }

    do {
        let episodes = try await service.episodes(of: season)
        // Another season may have been selected while these loaded.
        guard season == selectedSeason else {
            return
        }
        self.episodes = episodes
        self.episodesError = nil
    } catch { … }   // as in any load, recording an error only while its season is selected
}
```

- Changing the key is a synchronous action that empties the data, clears its error and paging, and sets `isLoading…`, since its load follows at once. The old key's data never shows under the new one, and the first frame shows progress rather than the empty state.
- `XView` loads from `.task(id: viewModel.selectedSeason)`. SwiftUI cancels the previous key's load, and loads again when the view reappears.
- A response can arrive before its cancellation does, so a load whose key no longer matches drops its result or error and leaves `isLoading…` to the current key's load.
- Typed text is input, not a key. The view model holds it as a plain `var` the field binds to, and `XView` applies it to the key after a pause in typing (`.task(id: viewModel.text)` with a short sleep), and at once on submit or when the field is cleared.

## Focused models and modes

When one part of a screen has its own lifecycle, give it its own `@Observable` model so only that part re-renders. The app's `XScreen` builds it with the screen's service, or the port it needs, and `XView` owns it beside the view model. Name it by one test, whether it would make sense without the views it drives:
- No: it holds presentation state for one part of the screen, such as its input, loading, errors and paging. It's a view model, named after the views it drives: `GenreDetailSearchViewModel` drives `GenreDetailSearchStateView`.
- Yes: it's a concept of its own that other screens could use. It's a plain noun, with no SwiftUI term of art: `WatchlistMembership`.

```swift
/// Whether the movie on screen is in the watchlist. Loads when its button appears.
@MainActor
@Observable
public final class WatchlistMembership {
    private let movieID: MovieSummary.ID
    private let service: any WatchlistMembershipService

    private(set) var isInWatchlist = false
    private(set) var isUpdating = false

    public init(movieID: MovieSummary.ID, service: any WatchlistMembershipService) { … }

    func load() async { … }
    func toggle(_ movie: MovieSummary) async { … }
}
```

`MovieDetailView(viewModel:watchlistMembership:)` owns both; only `WatchlistButton` reads `WatchlistMembership`.

### Modes

A mode replaces a screen's content with its own while it's active, driven by its own input: typically a server search scoped to one screen, such as a genre's movies. It's the exception, so check the usual answers first:
- Search across the app is its own screen, usually its own feature, with its own service, view model and views.
- Searching what a screen already shows filters the loaded values in its content view, with no new resource.

A mode is a view model of its own, made with the screen's service. Its text is input bound to the search field, its key is that text once applied, and its results are a [keyed](#keyed-resources), paged resource:

```swift
/// A search in a genre, by the words in its search field: what the genre's "See all" shows while there are words.
@MainActor
@Observable
public final class GenreDetailSearchViewModel {
    private let genreID: Genre.ID
    private let service: any GenreDetailService

    /// The words in the search field.
    var text = ""
    /// The words the results are for: the field's, trimmed, once applied. Empty while not searching.
    private(set) var query = ""

    /// Results
    private(set) var results: [MovieSummary] = []
    private(set) var isLoadingResults: Bool = false
    private(set) var resultsError: GenreDetailError?
    // nextResultsPage, isLoadingNextResultsPage, nextResultsPageError

    var isActive: Bool {
        !query.isEmpty
    }

    /// Searches for the words in the field. New words start the results over, and load from `load()`.
    func apply() { … }
    func load() async { … }
    func loadNextResultsPage() async { … }
}
```

- `GenreDetailService` gains `movies(genreID:matching:page:)`; the mode doesn't get a service of its own.
- `XView` owns both view models, and the screen's state view picks the mode in its first branch ([views](views.md#modes)).

## Testing view models

Swift Testing, `@MainActor` suites, one behavior per test, `@testable import XUI` to reach internal state and methods. A private actor stub in the test file returns queued results:

```swift
/// Returns each queued result in turn, one per call.
private actor StubMovieDetailService: MovieDetailService {
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

Cover the rules the screen follows: a resource loaded on appear starts out loading, the data loads, the error is recorded, a failed refresh keeps the data, a successful retry clears the error, and cancellation isn't an error. A paged screen also covers a failed page keeping what's loaded, a retried page loading, the first page clearing the next page's error, a page that arrives after a refresh being dropped, and a restarted footer still loading its page. Tests that interleave two loads use a stub that holds each request until the test answers it.

A keyed resource also covers a new key starting the data over and showing it as loading, and an old key's load being dropped and leaving the current key loading. A mode's view model is tested on its own, like any view model.
