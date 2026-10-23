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
- Nested models and modes
- Testing view models

## The shape

```swift
/// Everyone credited in a movie.
@MainActor
@Observable
public final class MovieCastViewModel {
    private let movieID: MovieSummary.ID
    private let service: any MovieCastService

    /// Cast
    private(set) var cast: [CastMember] = []
    private(set) var isLoadingCast: Bool = true
    private(set) var castError: MovieError?

    public init(movieID: MovieSummary.ID, service: any MovieCastService) {
        self.movieID = movieID
        self.service = service
    }

    func load() async {
        isLoadingCast = true
        defer { isLoadingCast = false }

        do {
            let cast = try await service.cast(movieID: movieID)
            self.cast = cast
            self.castError = nil
        } catch is CancellationError {

        } catch let error as MovieError {
            self.castError = error
        } catch {
            self.castError = .unavailable
        }
    }
}
```

- `@MainActor @Observable public final class`, one per screen: a view pushed onto a stack, or a tab's root. Never one for a part of a screen ([nested models](#nested-models-and-modes)).
- `public init` takes IDs and services and only assigns: the app builds a view model on every `body` pass and the view keeps the first, so `init` starts no tasks, subscriptions or I/O.
- State is `private(set)`. Actions are internal imperative verbs the views call: `load()`, `refresh()`, `loadNextShelvesPage()`, `toggle(_:)`. Three kinds of state are plain `var`s instead: input a view binds (a search field's `text`), what the screen presents (`isFilterPresented`), and what a sheet edits before it's applied (`filterDraft`) ([presentations](navigation.md#sheets-covers-and-alerts)).
- It stores what the service returns as it is: no data processing, pushing or routing, views, display strings or other features.
- `load()` is safe to call again, for a retry or the view reappearing, with no `loadIfNeeded()` variant or "already loaded" flag. A screen with a paged list is the one exception ([coming back to a paged list](#coming-back-to-a-paged-list)).

## One resource: data, activity and error

A screen can have data while it refreshes, data and a failed refresh, or an error while it retries. One `idle / loading / loaded / failed` enum holds only one of these at a time, so each resource gets three plain properties instead, with no shared state type:

| Property | Holds |
|---|---|
| The data, named for what it is (`cast`, `movie`) | An empty collection, or `nil` for a single value, until the first successful load; a failed refresh keeps it |
| `isLoading…` (`isLoadingCast`) | Whether its load is running. It starts `true` for a resource the screen loads as it appears, so the first frame shows progress rather than the empty state |
| `…Error` (`castError`) | The error its views switch over (`MovieError?`) |

- A load sets `isLoading…` for its duration (`defer`). On success it assigns what the service returns and clears the error; on failure it keeps the data and records the error, so a retry keeps the error on screen until a load succeeds.
- A resource keeps only the state its views read. One the screen quietly leaves out when it fails, such as Discover's featured movies, keeps just its data: its load assigns on success and does nothing on failure.

## Errors and cancellation

Each error property has its own type, usually the feature's `XError` ([domain](domain.md#errors)). Catch in this order:
1. `CancellationError`: nothing happened. Keep the data and the error; only the `defer` ends `isLoading…`.
2. `let error as XError`: record it.
3. Anything else: the fallback case of that property's type (`.unavailable`).

Cancellation is never a failure. The networking client turns a cancelled request into `CancellationError` ([networking](networking.md#errors-and-cancellation)), and `XView` runs `load()` from `.task`, which cancels it when the view disappears and runs it again when it reappears ([views](views.md#xview-xstateview-and-xcontentview)). The second run is a silent refresh that keeps the data on screen.

- A retry button starts `Task { await viewModel.load() }`, keeping its main-actor start, since `load()` begins by setting state. The task isn't cancelled with the view, which is harmless because `load()` can run again.
- Overlapping loads, such as a retry during a refresh, ask for the same data, so the last to finish wins.

How tasks and isolation work is a Swift Concurrency skill's subject ([related skills](../SKILL.md#related-skills)).

## Several resources

When a screen loads more than one resource, each gets its own properties and load method. The state view follows the resource the content can't do without; the others show when they arrive and are left out when they fail. Discover shows featured movies over a paged list of shelves, and follows the shelves:

```swift
public protocol DiscoverService: Sendable {
    /// - Throws: `DiscoverError`. Cancellation propagates as `CancellationError`.
    func featured() async throws -> [MovieSummary]

    /// - Throws: `DiscoverError`. Cancellation propagates as `CancellationError`.
    func shelves(page: Int) async throws -> Page<DiscoverShelf>
}

/// Featured movies over shelves of movies, with the shelves paged in as the last one comes into view.
@MainActor
@Observable
public final class DiscoverViewModel {
    private let service: any DiscoverService

    /// Featured
    private(set) var featured: [MovieSummary] = []

    /// Shelves
    private(set) var shelves: [DiscoverShelf] = []
    private(set) var isLoadingShelves: Bool = true
    private(set) var shelvesError: DiscoverError?
    private(set) var nextShelvesPage: Int?
    private(set) var isLoadingNextShelvesPage = false
    private(set) var nextShelvesPageError: DiscoverError?

    public init(service: any DiscoverService) {
        self.service = service
    }

    /// Loads unless there are shelves, so coming back to Discover keeps the pages it has.
    func load() async {
        guard shelves.isEmpty else {
            return
        }

        await refresh()
    }

    /// Loads the featured movies again, and the shelves from the first page.
    func refresh() async {
        await withDiscardingTaskGroup { group in
            group.addTask {
                await self.loadFeatured()
            }

            group.addTask {
                await self.loadShelves()
            }
        }
    }

    // loadShelves() follows the one-resource shape and asks for page 1. loadFeatured() only assigns.
    // loadNextShelvesPage() is below.
}
```

- Independent loads run side by side in a discarding task group. A resource that depends on another, such as a list picked from the loaded movie, loads after it.
- When a rule spans several resources, such as a screen that shows nothing until two of them load, add computed `isLoading` and `error` for the state view. Never store them: two loads would write one value.

## Pagination

A paged resource adds three properties to its data, `isLoading…` and `…Error`:

| Property | Holds |
|---|---|
| `next…Page` (`nextShelvesPage`) | The page to ask for next, from the service's `Page<Item>`; `nil` after the last one |
| `isLoadingNext…Page` | Whether the next page is loading |
| `next…PageError` | Why the next page failed, with its own type: a footer often shows fewer cases than the error view |

```swift
func loadNextShelvesPage() async {
    guard let requestedPage = nextShelvesPage else {
        return
    }

    isLoadingNextShelvesPage = true
    defer { isLoadingNextShelvesPage = false }

    do {
        let page = try await service.shelves(page: requestedPage)
        // A refresh may have started the shelves over while this page loaded.
        guard nextShelvesPage == requestedPage else {
            return
        }
        self.shelves.append(contentsOf: page.items)
        self.nextShelvesPage = page.nextPage
        self.nextShelvesPageError = nil
    } catch is CancellationError {

    } catch let error as DiscoverError {
        self.nextShelvesPageError = error
    } catch {
        self.nextShelvesPageError = .unavailable
    }
}
```

- Loading the first page replaces the list, sets `next…Page` and clears `next…PageError`.
- A later page appends, and is dropped when the list no longer needs that page next: a refresh started it over, or a duplicate call already appended it.
- No in-flight guard: the footer's task restarts as the footer scrolls out and back in, and the restarted call must be able to load.
- No de-duplication across pages and no merging helpers on models; the service decides what a page holds.

### Coming back to a paged list

Coming back to a screen runs its `.task` again. Loading the first page again would replace every page the user scrolled through, and the shorter list would bring the footer on screen and load page two at once. So a screen with a paged list guards `load()` on that list and moves its loads into `refresh()`, as [Discover does](#several-resources):

- `.refreshable` calls `refresh()`, which starts the list over and loads the screen's other resources again.
- A retry calls `load()`: after a failure the list is empty, so the guard lets it through.
- A resource beside the list, such as the featured movies, loads again only on a refresh.

## A thing and its paged list

A screen that shows one thing and a paged list belonging to it, such as a genre and its movies, loads them as two resources from the screen's own service:

```swift
/// A genre and its movies, a page at a time.
public protocol GenreDetailService: Sendable {
    func genre(id: Genre.ID) async throws -> Genre
    func movies(genreID: Genre.ID, page: Int) async throws -> Page<MovieSummary>
}

@MainActor
@Observable
public final class GenreDetailViewModel {
    /// Genre
    private(set) var genre: Genre?

    /// Movies
    private(set) var movies: [MovieSummary] = []
    private(set) var isLoadingMovies: Bool = true
    private(set) var moviesError: GenreDetailError?
    private(set) var nextMoviesPage: Int?
    private(set) var isLoadingNextMoviesPage = false
    private(set) var nextMoviesPageError: GenreDetailError?

    // load() guards on movies; refresh() loads the genre and the first page side by side.
}
```

- The thing leaves the list out, so nothing holds two copies of it. Don't page a value that contains the list, or pair a page with the thing in a wrapper type or tuple.
- Load the list after the thing only when the thing decides whether there is one; otherwise load both side by side.
- The state view follows the list, and the thing titles the screen from `XView`, so the title stays up in every state.

## Keyed resources

Some resources depend on a key the user changes on the screen: the words in a search, a selected department, a filter. The key is view model state, and the resource starts over whenever it changes:

```swift
/// Shows the person's credits in `department`. They start over, and load from `loadCredits()`.
func select(_ department: Department) {
    guard department != selectedDepartment else {
        return
    }

    selectedDepartment = department
    credits = []
    creditsError = nil
    // The view loads them next, so until then they show as loading rather than as none.
    isLoadingCredits = true
}

func loadCredits() async {
    let department = selectedDepartment
    isLoadingCredits = true
    // A load for a department no longer selected leaves the flag to the selected one's load.
    defer {
        if department == selectedDepartment {
            isLoadingCredits = false
        }
    }

    do {
        let credits = try await service.credits(personID: personID, department: department)
        // Another department may have been selected while these loaded.
        guard department == selectedDepartment else {
            return
        }
        self.credits = credits
        self.creditsError = nil
    } catch { … }   // as in any load, recording an error only while its department is selected
}
```

- Changing the key is synchronous: it empties the data, clears its error and paging, and sets `isLoading…` when a load follows. The old key's data never shows under the new one.
- A control that picks the key binds it through the action, so the reset always runs: `Picker(selection: Binding(get: { viewModel.selectedDepartment }, set: { viewModel.select($0) }))`.
- `XView` loads from `.task(id: viewModel.selectedDepartment)`. SwiftUI cancels the old key's load, but a response can arrive before its cancellation, so a load whose key no longer matches drops its result.
- A keyed paged list, such as search results, guards its load on the list like a [screen's](#coming-back-to-a-paged-list), and a new key empties the list, so the new key's load gets through.
- Typed text is input, not a key: a plain `var` the field binds. `XView` applies it to the key after a pause in typing (`.task(id: viewModel.text)` with a short sleep), and at once on submit or when the field is cleared.

## Nested models and modes

A part of a screen whose state changes on its own, such as a watchlist button or a search within the screen, gets a nested `@Observable` model: a type declared inside the screen's view model, made in its `init` with the services the view model was given, and owned by it as a `let`.

```swift
@MainActor
@Observable
public final class MovieDetailViewModel {
    // The movie and its cast, as in any view model.

    /// Whether the movie is in the watchlist; only the watchlist button reads it.
    let watchlistMembership: WatchlistMembership

    public init(
        movieID: MovieSummary.ID,
        service: any MovieDetailService,
        watchlistMembershipService: any WatchlistMembershipService
    ) {
        self.movieID = movieID
        self.service = service
        self.watchlistMembership = WatchlistMembership(movieID: movieID, service: watchlistMembershipService)
    }
}

extension MovieDetailViewModel {
    /// Whether the movie on screen is in the watchlist. Loads when its button appears.
    @MainActor
    @Observable
    final class WatchlistMembership {
        private(set) var isInWatchlist = false
        private(set) var isUpdating = false

        init(movieID: MovieSummary.ID, service: any WatchlistMembershipService) { … }

        func load() async { … }
        func toggle(_ movie: MovieSummary) async { … }
    }
}
```

- Only the views that read a nested model redraw when it changes: the watchlist button redraws for the watchlist, the rest of the screen doesn't.
- It uses its view model's services, the screen's own or a port, and never gets one of its own for being nested.
- Nest a part only some views read that has its own input, loading or errors. What the content shows stays on the view model, a key and what it loads included (a selected department and its credits), and so does a single flag or value such as `isFilterPresented`.
- A view binds a nested model's input through a binding it derives in `body`: `@Bindable var searchSession = viewModel.searchSession`, then `$searchSession.text`.

### Naming nested models

Name it for the one thing a user would recognize on the screen, as a plain noun that reads in "the screen has a ___":

| What it holds | Noun | Examples |
|---|---|---|
| Input that starts and ends a stretch of work, and what it loads | `…Session` | `SearchSession` |
| A fact about the item on screen that the user can change | the fact | `WatchlistMembership`, `MovieRating` |
| Something that runs over time | what runs | `TrailerPlayback` |
| Edits kept until they're saved | `…Draft` | `ReviewDraft` |

Never `…ViewModel` (screens own that), `…Manager`, `…Controller`, `…Handler`, `…Coordinator` or `…Helper`, `…State` or `…Store`.

### Modes

A mode replaces a screen's content with its own while it's active, driven by its own input: typically a server search scoped to one screen, such as a genre's movies. Search across the app is its own screen instead, and searching what a screen already shows filters the loaded values in its content view.

A mode is a nested model made with the screen's service. Its text is input bound to the search field, its key is that text once applied, and its results are a keyed, paged resource:

```swift
extension GenreDetailViewModel {
    /// A search in the genre: what the genre's screen shows while there are words to search for.
    @MainActor
    @Observable
    final class SearchSession {
        /// The words in the search field.
        var text = ""
        /// The words the results are for, once applied. Empty while not searching.
        private(set) var query = ""

        /// Results
        private(set) var results: [MovieSummary] = []
        private(set) var isLoadingResults: Bool = false
        private(set) var resultsError: GenreDetailError?
        // nextResultsPage, isLoadingNextResultsPage, nextResultsPageError

        var isActive: Bool {
            !query.isEmpty
        }

        func apply() { … }
        func load() async { … }
        func refresh() async { … }
        func loadNextResultsPage() async { … }
    }
}
```

`GenreDetailService` gains `movies(genreID:matching:page:)` for it, and the screen's state view picks the mode in its first branch ([views](views.md#modes)).

## Testing view models

Swift Testing, `@MainActor` suites, `@testable import XUI`, and a private actor stub of the service in each test file. Tests that interleave two loads use a stub that holds a request until the test releases it.

Test the rules a bug would break on this screen, not the shape every view model shares:
- What a load publishes, and what it asks the service for.
- The screen's own failure rule, such as Discover still showing its shelves when the featured movies fail.
- Paging: a next page appends, and loading again keeps the pages.
- Stale results: a page that arrives after a refresh, or a key's results after a new key, is dropped, and a restarted footer still loads its page.
- A key or a mode's rules: a new key starts the data over and shows it as loading.

Skip what the shape guarantees everywhere: starting values, the fallback error, a retry clearing an error, and cancellation alone, which the stale-result tests already cover. Test a nested model on its own, made the way its view model makes it.
