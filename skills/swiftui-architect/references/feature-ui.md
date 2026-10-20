# Feature UI

What lives in an `XUI` target and the shape of each piece. Keep feature views plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique beyond these shapes, use a SwiftUI skill.

## Contents
- File roles
- XView, XStateView and XContentView
- View models
- Focused models
- Ownership
- Loading and pagination: a proposed shape
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model as `@State`; applies screen modifiers; renders `XStateView` |
| `XStateView` | Shows the content, the error or progress, in that order |
| `XContentView` | Lays out the content in its scroll container, one view per resource or section |
| `XHeader`, `XRail`, `XRow`, `XCard`, `XBannersView`, `XSectionsView` | One section, resource or element each, taking only the values it reads |
| `X…NextPageFooter` | Loads the next page of a paged resource while it's on screen; takes the view model |
| `XErrorView` | Switches over `XError` and renders DesignSystem's `StateContent` with a retry; shared by all the feature's screens |
| `XToolbar` | A `ToolbarContent` struct; an item that opens a screen pushes an `XDestination` value |
| `XViewModel` | Loads and exposes state for one screen |
| Focused models | One concern of a screen, such as `WatchlistMembership` |
| `XDestination` | Where the feature's views can take the user |
| `Models/X+Title.swift` | Display mappings from XCore values to `LocalizedStringResource` |

A feature exports one public view per screen it owns (`MovieDetailView`, `MovieCastView`) and its view models. Everything else stays internal.

## XView, XStateView and XContentView

Each is its own file.

```swift
// DiscoverView.swift
public struct DiscoverView: View {
    @State private var viewModel: DiscoverViewModel

    /// The view takes ownership of `viewModel` when it first appears; later values are ignored.
    public init(viewModel: DiscoverViewModel) {
        self.viewModel = viewModel
    }

    public var body: some View {
        DiscoverStateView(viewModel: viewModel)
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

// DiscoverStateView.swift
/// Renders the shelves whenever there are some, the error only when there's nothing to show, and progress while the
/// first load runs. A retry loads again while the error stays on screen.
struct DiscoverStateView: View {
    let viewModel: DiscoverViewModel

    var body: some View {
        if !viewModel.shelves.isEmpty {
            DiscoverContentView(shelves: viewModel.shelves)
        } else if let error = viewModel.shelvesError {
            DiscoverErrorView(error: error, retry: retry)
        } else if viewModel.isLoadingShelves {
            ProgressView()
                .progressViewStyle(.loadingDots)
        } else {
            StateContent(.discoverEmptyTitle)
        }
    }

    private func retry() {
        Task { await viewModel.load() }
    }
}

// DiscoverContentView.swift
/// The shelves, one rail each.
struct DiscoverContentView: View {
    let shelves: [DiscoverShelf]

    var body: some View {
        ScrollView {
            LazyVStack(alignment: .leading, spacing: DesignSystem.Spacing.space7) {
                ForEach(shelves) { shelf in
                    DiscoverRail(shelf: shelf)
                }
            }
            .padding(.vertical, DesignSystem.Spacing.space5)
        }
    }
}
```

`XStateView` only picks what to show. It hands the error view its own `retry` method rather than an inline closure, so each branch stays one line and the action has a name. `XContentView` owns the scroll container and lays out the content, one view per resource or section. With one resource it takes the data as a value (`DiscoverContentView(shelves:)`); with several it takes the view model, because a view such as a paging footer needs it (see [Several resources and pagination](#several-resources-and-pagination)).

Anything that needs loaded data, such as a title from the model or a toolbar that only makes sense with content, is applied in `XContentView` (`.navigationTitle(movie.title)`), so `XView` never reads the data.

## View models

```swift
@MainActor
@Observable
public final class DiscoverViewModel {
    private let service: any DiscoverServiceProtocol

    /// Shelves
    private(set) var shelves: [DiscoverShelf] = []
    private(set) var isLoadingShelves: Bool = true
    private(set) var shelvesError: DiscoverError?

    public init(service: any DiscoverServiceProtocol) {
        self.service = service
    }

    func load() async {
        isLoadingShelves = true
        defer { isLoadingShelves = false }

        do {
            let shelves = try await service.loadShelves()
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
| State and content views, sections | Read what they need | Plain `let viewModel` or narrow values; Observation tracks each property read |
| A view that needs a binding | Derives one | `@Bindable var viewModel = viewModel` in `body`, or `$viewModel.query` from the owner |
| Presentation flags, form input, focus | The view that presents or edits | `@State private var` |

Pass views that only display the narrowest values they read (`title: String`, `movie: MovieSummary`, `sections: [HomeSection]`), not the whole view model or a large model with arrays. A view takes the view model only when it reads state of its own, such as a resource's loading flag or error, or calls the view model, like a paging footer. Don't pass closures down through views for the view model's actions instead: a closure can't be compared, so a view that takes one is re-evaluated whenever its parent is, while a view holding the view model re-renders only for the properties it reads. The exception is a leaf whose button fires one action, such as `XErrorView`'s retry: pass it a private method of the view that makes the call (`retry: retry`), not an inline closure.

## Loading and pagination: a proposed shape

This is a suggestion for keeping loading, errors and paging simple, not a rule. Adapt it when a screen needs more, such as a refresh indicator or a banner for a failed refresh. Every view model uses the same shape, whether its screen loads one resource or several, paged or not.

**Keep the data, the error and the activity apart, for each resource.** A screen can have data while it refreshes, data and a failed refresh, or an error while it retries. A single `idle / loading / loaded / failed` enum holds only one of these at a time, so each view model ends up rebuilding the others. Three plain properties per resource hold them all, with no shared state type:

| Property | Holds |
|---|---|
| The data, named for what it is (`shelves`, `movie`) | An empty collection, or `nil` for a single value, until the first successful load; a failed refresh keeps it |
| `isLoading…` (`isLoadingShelves`) | Whether its load is running. A resource the screen loads as soon as it appears starts `true`, so the first frame shows progress rather than the empty state |
| `…Error` (`shelvesError`) | An error the views switch over (`DiscoverError?`). Each error property has its own type: often the feature's `XError`, but a next page or one section can use another when its view shows different cases |

Rules for each load method:
- Set `isLoading…` for the duration of the load (`defer`).
- On success, assign what the service returns and clear the error. On failure, keep the data and record the error, so a retry keeps the error view on screen until a load succeeds.
- Cancellation is never a failure: catch `CancellationError` and change nothing.
- Unknown errors become the fallback case of that property's error type.

**Views render data first**: the content whenever there is data, the error view only when there is nothing to show, progress while the first load runs, and the empty state after it. A failed refresh keeps the content on screen without extra code.

### Several resources and pagination

When a screen loads more than one resource, or pages one, each resource gets its own properties and load method, and the screen gets computed `isLoading` and `error` for its state view. Home below loads its banners and a paged list of sections.

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
- A paged resource adds `next…Page`, `isLoadingNext…Page` and `next…PageError` to its data, `isLoading…` and `…Error`. `next…PageError` has its own type, which needn't match `…Error`: a footer often shows fewer cases than the error view. Home uses `HomeError` for both, but nothing in the shape depends on that.
- The screen-wide `isLoading` and `error` are computed from the resources and hold the screen's own rules, such as banners not failing Home. Never store them: two loads would end up writing one flag. `error` has the type the screen's error view switches over, which needn't be every resource's error type.
- `load()` runs the screen's independent loads side by side, each following the rules above. Loading the first page again also clears the next page's error.
- `loadNext…Page()` remembers the page it asked for and, after the `await`, drops the result if a refresh started the list over meanwhile. It has no in-flight guard: the footer's task restarts as the footer scrolls out and back in, and the restarted call must be able to load.
- When a resource depends on another, such as episodes picked from the loaded title, `load()` loads that one first and then the rest side by side. Start `load()` from `.task` on `XView`, not on the progress view: the progress view goes away when the first resource arrives, and cancelling its task would cancel the loads that follow.

The state view branches on the screen-wide properties. The content view lays out one view per resource: `HomeBannersView` and `HomeSectionsView` take their arrays, and only the footer, which reads its own paging state and loads the next page while it's on screen, takes the view model:

```swift
// HomeStateView.swift
/// Renders the error or progress while Home has nothing to show, and its content otherwise.
struct HomeStateView: View {
    let viewModel: HomeViewModel

    var body: some View {
        if let error = viewModel.error {
            HomeErrorView(error: error, retry: retry)
        } else if viewModel.isLoading {
            ProgressView()
                .progressViewStyle(.loadingDots)
        } else if viewModel.sections.isEmpty, viewModel.banners.isEmpty {
            StateContent(.homeEmptyTitle)
        } else {
            HomeContentView(viewModel: viewModel)
        }
    }

    private func retry() {
        Task { await viewModel.load() }
    }
}

// HomeContentView.swift
/// The banners above the sections, with the next page loading as the end comes into view.
struct HomeContentView: View {
    let viewModel: HomeViewModel

    var body: some View {
        ScrollView {
            VStack(spacing: DesignSystem.Spacing.space5) {
                if !viewModel.banners.isEmpty {
                    HomeBannersView(banners: viewModel.banners)
                }
                HomeSectionsView(sections: viewModel.sections)
                HomeSectionsNextPageFooter(viewModel: viewModel)
            }
        }
    }
}

// HomeSectionsNextPageFooter.swift
/// Loads the next page while the end of the sections is on screen, or offers a retry after a failure. Draws nothing
/// once the last page has loaded.
struct HomeSectionsNextPageFooter: View {
    let viewModel: HomeViewModel

    var body: some View {
        if viewModel.nextSectionsPageError != nil, !viewModel.isLoadingNextSectionsPage {
            Button(.homeActionRetry) {
                Task { await viewModel.loadNextSectionsPage() }
            }
            .buttonStyle(.filled)
        } else if let nextPage = viewModel.nextSectionsPage {
            HomeSectionsNextPageLoader(nextPage: nextPage, viewModel: viewModel)
        }
    }
}

private struct HomeSectionsNextPageLoader: View {
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
- No wrapper struct around a single system view, such as a `TabLabel` around `Label` or an icon view around one `Image`. Use the native initializers (`Label(_:image:)`, `Button(_:image:action:)`), and extract a struct only when it holds real layout or logic.
- No `Group { if … }` or `Group { switch … }` in a body: move the branching into an `XStateView` and apply modifiers outside it.
- Toolbars are `ToolbarContent` structs in XUI (`WatchlistToolbar`). An item that opens another screen pushes one of the feature's `XDestination` values, like any other link.
- Never create views from non-view types. View models expose XCore values; display mappings are XUI extensions returning `LocalizedStringResource` (`DiscoverCategory.title`), never `Text` or `Image`.
- Use native controls with DesignSystem styles: `Button` with `.buttonStyle(.filledProminent)`, `NavigationLink` with `.buttonStyle(.content)`. Inside a component that styles its own actions, such as `StateContent`, leave the style off.
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
