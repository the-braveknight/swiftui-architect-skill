# Feature UI

What lives in an `XUI` target and the shape of each piece. Keep feature views plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique beyond these shapes, use a SwiftUI skill.

## Contents
- File roles
- XView and XContentView
- View models
- Focused models
- Ownership
- Loading and pagination: a proposed shape
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model as `@State`; applies screen modifiers; renders `XContentView` |
| `XContentView` | Shows the data, the error or progress, in that order |
| `XSections` | Lays out the loaded content from narrow sections |
| `XHeader`, `XRail`, `XRow`, `XCard` | One section or element each, taking only the values it reads |
| `XErrorView` | Switches over `XError` and renders DesignSystem's `StateContent` with a retry; shared by all the feature's screens |
| `XToolbar` | A `ToolbarContent` struct |
| `XViewModel` | Loads and exposes state for one screen |
| Focused models | One concern of a screen, such as `WatchlistMembership` |
| `XDestination` | Where the feature's views can take the user |
| `Models/X+Title.swift` | Display mappings from XCore values to `LocalizedStringResource` |

A feature exports one public view per screen it owns (`MovieDetailView`, `MovieCastView`) and its view models. Everything else stays internal.

## XView and XContentView

```swift
public struct DiscoverView: View {
    @State private var viewModel: DiscoverViewModel

    /// The view takes ownership of `viewModel` when it first appears; later values are ignored.
    public init(viewModel: DiscoverViewModel) {
        self.viewModel = viewModel
    }

    public var body: some View {
        DiscoverContentView(viewModel: viewModel)
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .background(DesignSystem.Colors.Background.canvas)
            .navigationTitle(.discoverNavigationTitle)
            .task {
                await viewModel.load()
            }
            .refreshable {
                await viewModel.load()
            }
    }
}

/// Renders the shelves whenever there are some, and the error only when there's nothing to show.
struct DiscoverContentView: View {
    let viewModel: DiscoverViewModel

    var body: some View {
        if let shelves = viewModel.shelves {
            ScrollView {
                DiscoverRailList(shelves: shelves)
            }
        } else if let failure = viewModel.failure {
            DiscoverErrorView(error: failure) {
                Task { await viewModel.load() }
            }
        } else {
            ProgressView()
                .progressViewStyle(.loadingDots)
        }
    }
}
```

Anything that needs loaded data, such as a title from the model or a toolbar that only makes sense with content, is applied inside the data branch (`.navigationTitle(movie.title)`), so `XView` never reads the data.

## View models

```swift
@MainActor
@Observable
public final class DiscoverViewModel {
    private let service: any DiscoverServiceProtocol

    /// The shelves last loaded. A failed refresh keeps them.
    private(set) var shelves: [DiscoverShelf]?
    /// Why the last load failed.
    private(set) var failure: DiscoverError?

    public init(service: any DiscoverServiceProtocol) {
        self.service = service
    }

    func load() async {
        failure = nil
        do {
            shelves = try await service.loadShelves()
        } catch is CancellationError {
            // The view went away; not a failure.
        } catch {
            failure = error as? DiscoverError ?? .unavailable
        }
    }
}
```

Rules:
- `@MainActor @Observable final class`, `public init` taking IDs and service protocols, cheap (assignments only).
- State is `private(set)`; actions are internal methods the views call.
- `load()` calls one ready-made service method and assigns the result. Aggregation, concurrency and de-duplication belong in the service.
- `load()` is safe to call again (refresh, retry); no `loadIfNeeded()` variants or "already loaded" latches.
- No navigation, no views, no display strings, no other feature. State holds XCore values; display mappings are XUI extensions.

## Focused models

When one part of a screen has its own lifecycle, give it its own `@Observable` model so only that part re-renders. Name it as a noun for what it is, without the `ViewModel` suffix.

```swift
/// Whether the movie on screen is in the watchlist. Loads when its button appears.
@MainActor
@Observable
public final class WatchlistMembership {
    private let movieID: MovieSummary.ID
    private let service: any WatchlistMembershipServiceProtocol

    private(set) var isInWatchlist = false
    private(set) var isUpdating = false

    public init(movieID: MovieSummary.ID, service: any WatchlistMembershipServiceProtocol) { … }

    func load() async { … }
    func toggle(_ movie: MovieSummary) async { … }
}
```

`MovieDetailView(viewModel:watchlistMembership:)` owns both; only `WatchlistButton` reads `WatchlistMembership`.

## Ownership

| Who | Holds | How |
|---|---|---|
| App `XScreen` | Creates view models and focused models | Inline in `body`, passed to `XView` |
| `XView` | Owns them for the screen's lifetime | `@State private var viewModel` |
| Content views, sections | Read what they need | Plain `let viewModel` or narrow values; Observation tracks each property read |
| A view that needs a binding | Derives one | `@Bindable var viewModel = viewModel` in `body`, or `$viewModel.query` from the owner |
| Presentation flags, form input, focus | The view that presents or edits | `@State private var` |

Pass sections the narrowest values they read (`title: String`, `movie: MovieSummary`), not the whole view model or a large model with arrays.

## Loading and pagination: a proposed shape

This is a suggestion for keeping loading, errors and paging simple, not a rule. Adapt it when a screen needs more, such as a refresh indicator or a banner for a failed refresh.

**Keep the data, the failure and the activity apart.** A screen can have data while it refreshes, data and a failed refresh, or an error while it retries. A single `idle / loading / loaded / failed` enum holds only one of these at a time, so each view model ends up rebuilding the others. Plain properties hold them all, with no shared state type:

| Property | Holds |
|---|---|
| The data, named for what it is (`movie`, `shelves`, `items`) | `nil` until the first successful load; a failed refresh keeps it |
| `failure` | The feature's own error (`MovieDetailError?`), so the error view switches over known cases |
| An activity flag (`isSearching`) | Only where "loading" must be told apart from "not started" |

Rules for `load()`:
- Clear `failure` when a load starts, so a retry shows progress again.
- Assign the data on success and leave it alone on failure.
- Cancellation is never a failure: catch `CancellationError` and change nothing.
- Map unknown errors into the feature's error before storing them.

**Views render data first**: the content whenever there is data, the error view only when there is nothing to show, progress otherwise. A failed refresh keeps the content on screen without extra code.

### Pagination

Pagination types belong to the feature that pages. Nothing generic goes into Core until a second feature pages the same way.

- The service returns one page and where the next one starts: `SearchResultsPage(movies:nextPage:)`, where `nextPage: Int?` is `nil` on the last page. XLive derives it from the API's page numbers, so the UI never sees page counts.
- A small value type collects the pages for one query: `SearchResults` with `query`, `movies`, `nextPage` and `append(_:)`. `append` skips repeated items so `ForEach` identifiers stay unique.
- The view model keeps `results: SearchResults?` and adds `nextPageFailure: SearchError?` beside `failure`. A failed later page keeps the results; a failed search replaces them.
- The pagination footer is the only caller of `loadNextPage()`. Its progress view runs `.task(id: nextPage)`, so a page loads when the footer scrolls into view, again after each page while it's still visible (even a page of repeats), and one at a time. Its retry button only clears `nextPageFailure`; the progress view that replaces it loads the page.
- The results list gets `.id(results.query)`, so a new query starts a new list at the top, with its own footer task.

```swift
struct SearchPaginationFooter: View {
    let nextPage: Int
    let viewModel: SearchViewModel

    var body: some View {
        if let failure = viewModel.nextPageFailure {
            SearchPageErrorView(error: failure) {
                viewModel.retryNextPage()
            }
        } else {
            ProgressView()
                .task(id: nextPage) {
                    await viewModel.loadNextPage()
                }
        }
    }
}
```

Other ecosystems settled on the same ideas: TanStack Query keeps data status and fetch status apart and checks data first, Android Paging keeps a load state per direction, and Riverpod's `AsyncValue` keeps the previous value through loading and errors.

## View structure rules

- Separate `View` structs for every section. No computed `some View` properties and no `@ViewBuilder` helper functions.
- No `Group { if … }` or `Group { switch … }` in a body: move the branching into an `XContentView` and apply modifiers outside it.
- Toolbars are `ToolbarContent` structs (`WatchlistToolbar`).
- Never create views from non-view types. View models expose XCore values; display mappings are XUI extensions returning `LocalizedStringResource` (`DiscoverCategory.title`), never `Text` or `Image`.
- Use native controls with DesignSystem styles: `Button` with `.buttonStyle(.filledProminent)`, `NavigationLink` with `.buttonStyle(.content)`.
- Feature-local presentations are `@State` in the presenting view.
- Strings come from the feature's String Catalog through generated symbols.

## Previews

Every public view has a preview built on mocks:

```swift
#Preview {
    NavigationStack {
        DiscoverView(viewModel: DiscoverViewModel(service: .mock))
    }
}

#Preview("Arabic") {
    NavigationStack {
        DiscoverView(viewModel: DiscoverViewModel(service: .mock))
    }
    .environment(\.locale, Locale(identifier: "ar"))
    .environment(\.layoutDirection, .rightToLeft)
}
```

A feature's previews are its isolated demo: the feature never needs the app to be seen.
