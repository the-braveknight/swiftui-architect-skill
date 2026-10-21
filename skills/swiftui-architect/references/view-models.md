# View models

A view model loads what one screen shows and exposes it as state. Services do the work (mapping, aggregation, de-duplication) and return view-ready domain values; the view model assigns them and manages loading, errors and paging.

## Contents
- The shape
- One resource: data, activity and error
- Errors and cancellation
- Several resources: computed screen state
- Pagination
- A thing and its paged list
- Focused models
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

- `@MainActor @Observable public final class`, one per screen, with a `public init` taking IDs and services. The app's `XScreen` builds one on every `body` pass and the view keeps only the first, so `init` does assignments only: no tasks, subscriptions or I/O.
- State is `private(set)`; actions are internal methods the views call (`load()`, `loadNextSectionsPage()`, `toggle(_:)`), named as imperative verbs because they change state.
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
- Views render data first: the content whenever there is data, the error only when there is nothing to show, and progress while the first load runs ([views](views.md#xview-xstateview-and-xcontentview)).

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

## Several resources: computed screen state

When a screen loads more than one resource, each gets its own properties and load method, and the screen gets computed `isLoading` and `error` for its state view. Home loads its banners and a paged list of sections:

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
    private(set) var isLoadingBanners: Bool = false
    private(set) var bannersError: HomeError?

    /// Sections
    private(set) var sections: [HomeSectionSummary] = []
    private(set) var isLoadingSections: Bool = true
    private(set) var sectionsError: HomeError?
    private(set) var nextSectionsPage: Int?
    private(set) var isLoadingNextSectionsPage = false
    private(set) var nextSectionsPageError: HomeError?

    /// Whether Home is still loading with nothing to show yet.
    var isLoading: Bool {
        isLoadingSections && sections.isEmpty
    }

    /// Why Home has nothing to show. Banners failing on their own don't count.
    var error: HomeError? {
        sections.isEmpty ? sectionsError : nil
    }

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

- The screen-wide `isLoading` and `error` are computed from the resources and hold the screen's own rules, such as banners not failing Home. Never store them: two loads would end up writing one flag. Only screens with several resources have them.
- `error` has the type the screen's state view switches over, which needn't be every resource's error type.
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

A screen that shows one thing and a paged list belonging to it, such as a Home section's "See all", loads them as two resources: the thing, then its list a page at a time. The screen has its own service:

```swift
/// A section's "See all": the section, then its movies a page at a time.
public protocol HomeSectionDetailService: Sendable {
    /// The section: its title and what it holds, without its movies.
    func section(id: HomeSection.ID) async throws -> HomeSection
    func movies(sectionID: HomeSection.ID, page: Int) async throws -> Page<MovieSummary>
}

@MainActor
@Observable
public final class HomeSectionDetailViewModel {
    private let sectionID: HomeSection.ID
    private let service: any HomeSectionDetailService

    /// Section
    private(set) var section: HomeSection?
    private(set) var isLoadingSection: Bool = true
    private(set) var sectionError: HomeSectionDetailError?

    /// Movies
    private(set) var movies: [MovieSummary] = []
    private(set) var isLoadingMovies: Bool = false
    private(set) var moviesError: HomeSectionDetailError?
    private(set) var nextMoviesPage: Int?
    private(set) var isLoadingNextMoviesPage = false
    private(set) var nextMoviesPageError: HomeSectionDetailError?

    /// Whether the detail has nothing to show yet: its section, or a section's first movies.
    var isLoading: Bool {
        (isLoadingSection && section == nil) || (isLoadingMovies && movies.isEmpty)
    }

    /// Why the detail has nothing to show. Unlike Home's feed, a section opened by its id can be not found.
    var error: HomeSectionDetailError? {
        section == nil ? sectionError : (movies.isEmpty ? moviesError : nil)
    }

    func load() async {
        await loadSection()
        // Only a section of movies pages them; any other section comes whole.
        if case .movies = section?.content {
            await loadMovies()
        }
    }

    // loadSection(), loadMovies() and loadNextMoviesPage() follow the shapes above.
}
```

- The thing leaves the list out, so nothing holds two copies of it. Don't page a value that contains the list, merge pages into it, or pair it with its next page in a wrapper type or tuple.
- When the API returns the thing with its first page, this costs an extra request. Accept it for two plain resources.
- Load the list after the thing only when the thing decides whether there is one; otherwise load both side by side.

## Focused models

When one part of a screen has its own lifecycle, give it its own `@Observable` model so only that part re-renders. Name it as a noun for what it is, without the `ViewModel` suffix and without a SwiftUI term of art.

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
