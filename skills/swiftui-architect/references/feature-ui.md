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
- A view model does no data processing of its own. It calls ready-made service methods, assigns what they return and manages the state its views read; mapping, aggregation and de-duplication belong in the service.
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

This is a suggestion for keeping loading, errors and paging simple, not a rule. Adapt it when a screen needs more, such as a refresh indicator or a banner for a failed refresh. A screen with one resource can use the shape below; a screen with several resources or pagination uses the one in [Several resources and pagination](#several-resources-and-pagination).

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

### Several resources and pagination

When a screen loads more than one resource, or pages one, each resource gets its own state and its own load method, and the screen gets computed `isLoading` and `error` for its content view. Home below loads its banners and a paged list of sections.

**Services stay this readable.** One method per resource, each returning values the views show as they are. XLive does all the mapping, aggregation and de-duplication; a paged method returns `Page<Item>`, a small generic struct in Core.

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

public protocol HomeServiceProtocol: Sendable {
    /// - Throws: `HomeError`. Cancellation propagates as `CancellationError`.
    func loadBanners() async throws -> [MovieSummary]

    /// - Throws: `HomeError`. Cancellation propagates as `CancellationError`.
    func loadSections(page: Int) async throws -> Page<HomeSection>
}

public struct HomeService: HomeServiceProtocol {
    private let client: TMDBClient

    public init(client: TMDBClient) {
        self.client = client
    }

    @concurrent
    public func loadBanners() async throws -> [MovieSummary] {
        do {
            let response = try await client.homeBanners()
            return (response.results ?? []).compactMap(MovieSummary.init)
        } catch let error as TMDBError {
            throw HomeError(error)
        }
    }

    @concurrent
    public func loadSections(page: Int) async throws -> Page<HomeSection> {
        do {
            let response = try await client.home(page: page)
            return Page(response)
        } catch let error as TMDBError {
            throw HomeError(error)
        }
    }
}

// XLive/Mapping/Page+TMDB.swift
extension Page where Item == HomeSection {
    /// The API numbers its pages; the feature only needs to know which one comes next.
    init(_ response: Components.Schemas.HomePage) {
        let number = response.page ?? 1
        let totalPages = response.totalPages ?? number
        self.init(
            items: (response.sections ?? []).compactMap(HomeSection.init),
            nextPage: number < totalPages ? number + 1 : nil
        )
    }
}
```

Live service methods are `@concurrent`, so their requests and mapping run off the main actor of the view model that awaits them.

**View models do no data processing of their own.** They only manage the state their views read: every value they store comes from the service as it is.

```swift
/// Home's banners and sections, with the sections paged in as the last one comes into view.
@MainActor
@Observable
public final class HomeViewModel {
    private let service: any HomeServiceProtocol

    /// Banners
    private(set) var banners: [MovieSummary] = []
    private(set) var isLoadingBanners: Bool = false
    private(set) var bannersError: HomeError?

    /// Sections
    private(set) var sections: [HomeSection] = []
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

    public init(service: any HomeServiceProtocol) {
        self.service = service
    }

    func load() async {
        await withTaskGroup { group in
            group.addTask {
                await self.loadBanners()
            }

            group.addTask {
                await self.loadSections()
            }
        }
    }

    func loadBanners() async {
        guard !isLoadingBanners else {
            return
        }

        isLoadingBanners = true
        defer { isLoadingBanners = false }

        do {
            let banners = try await service.loadBanners()
            self.banners = banners
            self.bannersError = nil
        } catch is CancellationError {

        } catch let error as HomeError {
            self.bannersError = error
        } catch {
            self.bannersError = .unavailable
        }
    }

    func loadSections() async {
        isLoadingSections = true
        defer { isLoadingSections = false }

        do {
            let page = try await service.loadSections(page: 1)
            self.sections = page.items
            self.nextSectionsPage = page.nextPage
            self.sectionsError = nil
            self.nextSectionsPageError = nil
        } catch is CancellationError {

        } catch let error as HomeError {
            self.sectionsError = error
        } catch {
            self.sectionsError = .unavailable
        }
    }

    func loadNextSectionsPage() async {
        guard let requestedPage = nextSectionsPage else {
            return
        }

        isLoadingNextSectionsPage = true
        defer { isLoadingNextSectionsPage = false }

        do {
            let page = try await service.loadSections(page: requestedPage)
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
}
```

Rules for this shape:
- Each resource has its data (empty until it loads), `isLoading…` and `…Error`. A paged resource adds `next…Page`, `isLoadingNext…Page` and `next…PageError`.
- A resource the screen loads as soon as it appears starts with `isLoading… = true`, so the first frame shows progress rather than the empty state.
- The screen-wide `isLoading` and `error` are computed from the resources and hold the screen's own rules, such as banners not failing Home. Never store them: two loads would end up writing one flag.
- `load()` runs the screen's independent loads side by side. Each load assigns what the service returns, clears its error on success and keeps its data on failure. Loading the first page again also clears the next page's error. Cancellation changes nothing, and unknown errors become the feature's fallback case.
- `loadNext…Page()` remembers the page it asked for and, after the `await`, drops the result if a refresh started the list over meanwhile. It has no in-flight guard: the footer's task restarts as the footer scrolls out and back in, and the restarted call must be able to load.
- When a resource depends on another, such as episodes picked from the loaded title, `load()` loads that one first and then the rest side by side. Start `load()` from `.task` on `XView`, not on the progress view: the progress view goes away when the first resource arrives, and cancelling its task would cancel the loads that follow.

The content view branches on the screen-wide properties, and a footer after the sections loads the next page while it's on screen:

```swift
struct HomeContentView: View {
    let viewModel: HomeViewModel

    var body: some View {
        if let error = viewModel.error {
            // In a scroll view, so pulling to refresh works here too.
            ScrollView {
                HomeErrorView(error: error) {
                    Task { await viewModel.load() }
                }
            }
        } else if viewModel.isLoading {
            ProgressView()
                .progressViewStyle(.loadingDots)
        } else if viewModel.sections.isEmpty, viewModel.banners.isEmpty {
            ScrollView {
                StateContent(.homeEmptyTitle)
            }
        } else {
            HomeSections(viewModel: viewModel)
        }
    }
}

private struct HomeNextPageLoader: View {
    let nextPage: Int
    let viewModel: HomeViewModel
    @State private var isVisible = false

    var body: some View {
        ProgressView()
            .onScrollVisibilityChange { isVisible = $0 }
            // Asks again after each page in case the footer is still on screen once the page arrives.
            .task(id: isVisible ? nextPage : nil) {
                if isVisible { await viewModel.loadNextSectionsPage() }
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
