# Navigation

Native SwiftUI navigation only: `NavigationStack(path:)`, `NavigationLink(value:)`, `.navigationDestination(for:)`, plus local `sheet`, `fullScreenCover` and `alert`. The app owns navigation state and decides which screen each value shows; features only say where their views can go.

## Contents
- The model
- XDestination (feature)
- SharedDestination (SharedUI)
- Router and TabDestination (app)
- Destination and DestinationView (app)
- Registering destinations
- Programmatic navigation
- Push or present
- Presenting a screen
- Sheets, covers and alerts
- Previews

## The model

```
Feature view ──── NavigationLink(value: CatalogDestination.cast(movieID: 550))
                        │  appended to the tab's NavigationPath
                        ▼
Tab XScreen ───── .navigationDestination(for: CatalogDestination.self)
                        │  Destination(destination)  → .movieCast(movieID: 550)
                        ▼
DestinationView ─ MovieCastScreen(movieID: 550) ── reads services, builds the view model
```

The same path works inside a feature (a movie's details → its cast) and across features (a genre's movies in Discover → a movie's details in Catalog). The feature never knows which.

- Feature views push values with `NavigationLink(value:)`, and never receive navigation closures, a router or a path. The one thing the app passes in is the screen an `XView` presents ([presenting a screen](#presenting-a-screen)).
- View models never push or route: they hold what the screen presents, and its views act on it. A screen that closes once an action succeeds, such as a form that saves, calls `dismiss()` from its view when the view model's state says the action is done.

## XDestination (feature)

One enum per feature in `XUI/Navigation/XDestination.swift`, however many tabs show it: everywhere its views can take the user, its own screens and other features' alike, in the feature's own words.

```swift
/// Where the Catalog feature's views can take the user, besides the places several features open (`SharedDestination`).
public enum CatalogDestination: Hashable, Codable, Sendable {
    /// Everyone credited in a movie.
    case cast(movieID: MovieSummary.ID)
    /// The person behind a credit.
    case person(id: Person.ID)
}

NavigationLink(.catalogMovieDetailCastSeeAll, value: CatalogDestination.cast(movieID: movieID))
```

- A case may open another feature's screen and is still named in this feature's words: Watchlist's `.signIn` opens Account's screen ([presenting a screen](#presenting-a-screen)). It carries a plain `Int` when the id's type belongs to the other feature, since features don't import each other.
- Cases carry identifiers, never models or data to display; the pushed screen loads what it shows. `Codable` keeps stacks restorable and deep-linkable.
- Name cases for the place, not the action (`.person(id:)`, not `.showPerson`).
- A feature whose every place is shared has no `XDestination`.

## SharedDestination (SharedUI)

The places several features open, such as a movie's details, are cased once, in `SharedUI/Navigation/SharedDestination.swift`:

```swift
/// Places several features open. The app maps each case to its screen once and registers this type in every stack.
public enum SharedDestination: Hashable, Codable, Sendable {
    /// A movie's details.
    case movieDetail(id: MovieSummary.ID)
}
```

- A place moves here once two features link to it. The feature that owns the screen counts only when its own views link to it. A place only one feature links to stays in that feature's `XDestination`, even when the screen belongs to another feature: only Watchlist links to signing in (`WatchlistDestination.signIn`).
- Cases name the screen they open (`.movieDetail(id:)`), since the enum sits outside every feature; a feature's `XDestination` names places in its own words instead (`.cast(movieID:)`).
- A case carries a Shared type's id where one exists (`MovieSummary.ID`), and a plain `Int` otherwise, since SharedUI imports no feature.
- A SharedUI view that links, such as `MovieSummaryCard`, pushes `SharedDestination` itself, with no protocol or closures for its links.

## Router and TabDestination (app)

```swift
enum TabDestination: Hashable {
    case discover
    case watchlist
    case search
}

/// The app's navigation state: the selected tab and each tab's stack.
@MainActor
@Observable
final class Router {
    var selectedTab: TabDestination = .discover
    private var paths: [TabDestination: NavigationPath] = [:]

    subscript(tab: TabDestination) -> NavigationPath {
        get { paths[tab] ?? NavigationPath() }
        set { paths[tab] = newValue }
    }
}
```

Inject it once from `<AppName>App` with `.environment(router)`. Each tab XScreen binds its stack to its own entry (`NavigationStack(path: $router[.discover])`), so a new tab adds a `TabDestination` case and nothing else to `Router`. `TabDestination` is named like every other place the app can go, which leaves SwiftUI's `Tab` free for `ContentView`.

## Destination and DestinationView (app)

`Destination` lists the screens features reach, pushed or presented. One initializer per destination type is the only place a destination becomes an app screen, so all cross-feature wiring reads in one file:

```swift
/// The screens features reach, pushed or presented. Features never see this type.
enum Destination: Hashable {
    case movieDetail(id: MovieSummary.ID)
    case movieCast(movieID: MovieSummary.ID)
    case genreDetail(id: Genre.ID)
    case personDetail(id: Person.ID)
    case signIn
}

extension Destination {
    /// The places several features open lead to the same screen from every tab.
    init(_ destination: SharedDestination) {
        switch destination {
        case .movieDetail(let id): self = .movieDetail(id: id)
        }
    }

    init(_ destination: DiscoverDestination) {
        switch destination {
        case .genre(let id): self = .genreDetail(id: id)
        }
    }

    init(_ destination: CatalogDestination) {
        switch destination {
        case .cast(let movieID): self = .movieCast(movieID: movieID)
        case .person(let id): self = .personDetail(id: id)
        }
    }

    init(_ destination: WatchlistDestination) {
        switch destination {
        case .signIn: self = .signIn
        }
    }
}

/// The screen for a `Destination`.
struct DestinationView: View {
    let destination: Destination

    var body: some View {
        switch destination {
        case .movieDetail(let id):
            MovieDetailScreen(movieID: id)
        case .movieCast(let movieID):
            MovieCastScreen(movieID: movieID)
        case .genreDetail(let id):
            GenreDetailScreen(genreID: id)
        case .personDetail(let id):
            PersonDetailScreen(personID: id)
        case .signIn:
            SignInScreen()
        }
    }
}
```

A feature can link to a place before its screen is built: `DestinationView` shows a placeholder until it ships, and only `DestinationView` changes then.

## Registering destinations

Every tab XScreen registers, at its stack root, `SharedDestination` and each `XDestination` type that can appear in its stack ([tab XScreen](composition.md#xscreen-tab-and-pushed)):

```swift
.navigationDestination(for: CatalogDestination.self) { destination in
    DestinationView(destination: Destination(destination))
}
```

- Register only at stack roots. A registration inside a feature screen runs again when that screen appears twice in one stack, such as a movie pushed from another movie, and SwiftUI reports a duplicate.
- The repetition across tabs is deliberate: each stack shows everything it can push.
- A link whose type isn't registered does nothing at runtime. When a feature becomes reachable from a new stack, register its destination there.

## Programmatic navigation

Only the app navigates in code, by changing Router state. Deep links and notifications resolve to a destination and append it:

```swift
router.selectedTab = .discover
router[.discover].append(SharedDestination.movieDetail(id: 550))
```

Pop to root by assigning an empty `NavigationPath()`. Since every destination is `Codable`, `NavigationPath.CodableRepresentation` can persist and restore stacks.

## Push or present

Whether a screen is pushed or presented is the design's choice. Who builds it isn't: a view that loads its own data or needs services is a screen, and only the app builds screens.

- **Push** (`NavigationLink(value:)`) a screen that belongs in the back stack, such as a movie's details.
- **Present a screen** (`sheet(item:)`, `fullScreenCover(item:)`) that the design shows over the current one, such as signing in. The app builds it too ([presenting a screen](#presenting-a-screen)).
- **Present a view** the feature builds (`sheet`, `alert`, `confirmationDialog`) when it's made from what the presenting screen holds: a full biography, a poster viewer, a confirmation, a filter it edits. It takes plain values, or reads the presenting screen's view model when it changes that screen's state. It never gets a service or a view model of its own.
- A presentation that changes data, such as the confirmation before removing a watchlist item, calls an action on the presenting view model, which makes the change through the screen's own service.

## Presenting a screen

A screen the design shows as a sheet or a full-screen cover, such as signing in from the Watchlist tab, is a place like any other: a case in the presenting feature's `XDestination`, mapped in `Destination.init(_:)` and built by `DestinationView`. The feature holds what's presented and picks how; the app supplies the screen:

```swift
// WatchlistUI/Navigation/WatchlistDestination.swift
/// Where the Watchlist feature's views can take the user, besides the places several features open.
public enum WatchlistDestination: Hashable, Codable, Sendable, Identifiable {
    /// Signing in, so the watchlist follows the account.
    case signIn

    public var id: Self { self }
}

// WatchlistViewModel
/// The place shown over the watchlist, while it's up.
var presentedDestination: WatchlistDestination?

// WatchlistView.swift
public struct WatchlistView<Presented: View>: View {
    @State private var viewModel: WatchlistViewModel
    private let presented: (WatchlistDestination) -> Presented

    /// `presented` builds the screen for each place the watchlist shows over itself.
    public init(viewModel: WatchlistViewModel, @ContentBuilder presented: @escaping (WatchlistDestination) -> Presented) {
        self.viewModel = viewModel
        self.presented = presented
    }

    public var body: some View {
        WatchlistPhaseView()
            // …
            .fullScreenCover(item: $viewModel.presentedDestination, content: presented)
            .environment(viewModel)
    }
}

// App: WatchlistScreen.swift
WatchlistView(viewModel: WatchlistViewModel(service: watchlistService)) { destination in
    NavigationStack {
        DestinationView(destination: Destination(destination))
            // A stack root of its own: it registers what it can push.
    }
}
```

- An action on the view model opens it (`openSignIn()`). The presented screen closes itself with `dismiss()`, from its close button or once its view model says it's done, which sets `presentedDestination` back to `nil`.
- The feature picks `sheet(item:)` or `fullScreenCover(item:)` for its `presentedDestination`; the app picks only the screen. Several places a screen presents share that one property and its style, so only one is up at a time. The same case can be pushed from one view and presented from another.
- Only `XView` takes the closure, and only when the feature presents a place; the views below it still push values.
- The presented screen's XScreen is shaped like a pushed one ([XScreen](composition.md#xscreen-tab-and-pushed)). The app wraps it in a `NavigationStack` when it needs a title, a close button or pushes of its own. That stack is a stack root, so it registers every destination type it can show ([registering destinations](#registering-destinations)).
- The feature's previews pass a placeholder or its preview destination view ([previews](#previews)).

## Sheets, covers and alerts

What a screen presents is state on its view model, a plain `var` its views bind, so any view can open it, an action can close it, and tests can check it. What a presented view edits before it's applied is a draft on the view model. The action that opens the sheet fills it, so each presentation starts from the screen's state:

```swift
// SearchViewModel
/// Whether the filter is open.
var isFilterPresented: Bool = false
/// The filter being edited while it's open, starting from the search's own each time it opens.
var filterDraft = SearchFilter()

/// Opens the filter on the search's own.
func openFilter() {
    filterDraft = query.filter
    isFilterPresented = true
}
```

- The toolbar's button calls `openFilter()`, `SearchFilterView` binds `$viewModel.filterDraft`, and its apply button calls `viewModel.apply(viewModel.filterDraft)`, which closes the filter.
- `XView` presents what any state can open, such as a toolbar's filter; a sheet only the content opens is presented there.
- A presented view that needs a title or a close button owns an unmanaged `NavigationStack`.
- A presentation about one value, such as the watchlist item a row asks to remove, is that value as an optional (`var itemToRemove: WatchlistItem?`). It shows while the value is set, its actions receive the value, and dismissing it sets the value back to `nil`. No `Bool` sits beside the value, since the two could disagree.
- A screen with several sheets holds one optional `Identifiable` enum of them (`presentedSheet`), never a `Bool` each, so only one is ever up.

## Previews

Feature previews wrap the view in a `NavigationStack`. To click through a feature's own screens, register its destinations on the preview root with a private builder that uses mocks:

```swift
#Preview {
    NavigationStack {
        MovieDetailView(viewModel: …)   // built on .mock services
            .navigationDestination(for: CatalogDestination.self) { destination in
                CatalogPreviewDestinationView(destination: destination)
            }
    }
}

/// Builds the Catalog feature's own screens on mock services for previews. The app builds the real ones.
private struct CatalogPreviewDestinationView: View {
    let destination: CatalogDestination

    var body: some View {
        switch destination {
        case .cast(let movieID):
            MovieCastView(viewModel: …)
        case .person(let id):
            PersonDetailView(viewModel: …)
        }
    }
}
```

A place that opens another feature's screen shows a placeholder in the builder (`Text(verbatim:)`), since only the app can build that screen.
