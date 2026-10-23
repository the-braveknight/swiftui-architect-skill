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
- Sheets, covers and alerts
- Previews

## The model

```
Feature view ──── NavigationLink(value: MovieDestination.cast(movieID: 550))
                        │  appended to the tab's NavigationPath
                        ▼
Tab XScreen ───── .navigationDestination(for: MovieDestination.self)
                        │  Destination(destination)  → .movieCast(movieID: 550)
                        ▼
DestinationView ─ MovieCastScreen(movieID: 550) ── reads services, builds the view model
```

The same path works inside a feature (a movie's details → its cast) and across features (Movie → Person). The feature never knows which.

- Feature views push values with `NavigationLink(value:)`, and never receive navigation closures, a router or a path.
- View models never navigate.

## XDestination (feature)

One enum per feature in `XUI/Navigation/XDestination.swift`, however many tabs show it: everywhere its views can take the user, its own screens and other features' alike, in the feature's own words.

```swift
/// Where the Movie feature's views can take the user, besides the places several features open (`SharedDestination`).
public enum MovieDestination: Hashable, Codable, Sendable {
    /// Everyone credited in a movie.
    case cast(movieID: MovieSummary.ID)
    /// The person behind a cast credit. Movie doesn't import Person, so the id is a plain `Int`.
    case person(id: Int)
}

NavigationLink(.movieDetailCastSeeAll, value: MovieDestination.cast(movieID: movieID))
```

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

- A place moves here once a second feature links to it. A place only one feature links to stays in its `XDestination`, even another feature's screen (`MovieDestination.person(id:)`).
- Cases name the screen they open (`.movieDetail(id:)`), since the enum sits outside every feature. A feature's `XDestination` names places in its own words (`.cast(movieID:)`).
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

Inject it once from `<AppName>App` with `.environment(router)`. Each tab XScreen binds its stack to its own entry (`NavigationStack(path: $router[.discover])`). `TabDestination` is named like every other place the app can go, which leaves SwiftUI's `Tab` free for `ContentView`.

## Destination and DestinationView (app)

`Destination` lists the app's pushed screens. One initializer per destination type is the only place a destination becomes an app screen, so all cross-feature wiring reads in one file:

```swift
/// The app's pushed screens. Features never see this type.
enum Destination: Hashable {
    case movieDetail(id: MovieSummary.ID)
    case movieCast(movieID: MovieSummary.ID)
    case genreDetail(id: Genre.ID)
    case personDetail(id: Person.ID)
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

    init(_ destination: MovieDestination) {
        switch destination {
        case .cast(let movieID): self = .movieCast(movieID: movieID)
        case .person(let id): self = .personDetail(id: id)
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
        }
    }
}
```

A feature can link to a place before its screen is built: `DestinationView` shows a placeholder until it ships, and only `DestinationView` changes then.

## Registering destinations

Every tab XScreen registers, at its stack root, `SharedDestination` and each `XDestination` type that can appear in its stack ([tab XScreen](composition.md#xscreen-tab-and-pushed)):

```swift
.navigationDestination(for: MovieDestination.self) { destination in
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

- **Push** (`NavigationLink(value:)`) when the screen loads its own data, needs services, other features can reach it, or it belongs in the back stack.
- **Present** (`sheet`, `fullScreenCover`, `alert`) when the content is self-contained and built from values the presenting view already has: a full biography, a poster viewer, a confirmation. A presented view takes plain values, or reads the presenting screen's view model when it changes that screen's state, such as a filter it edits; never a service.

## Sheets, covers and alerts

What a screen presents is state on its view model, a plain `var` its views bind, so any view can open it, an action can close it, and tests can check it. What a presented view edits before it's applied is a draft on the view model that the action opening the sheet fills, so each presentation starts from the screen's state:

```swift
// SearchViewModel
/// Whether the filter is open.
var isFilterPresented = false
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

## Previews

Feature previews wrap the view in a `NavigationStack`. To click through a feature's own screens, register its destinations on the preview root with a private builder that uses mocks:

```swift
#Preview {
    NavigationStack {
        MovieDetailView(viewModel: …)   // built on .mock services
            .navigationDestination(for: MovieDestination.self) { destination in
                MoviePreviewDestinationView(destination: destination)
            }
    }
}

/// Builds the Movie feature's own screens on mock services for previews. The app builds the real ones.
private struct MoviePreviewDestinationView: View {
    let destination: MovieDestination

    var body: some View {
        switch destination {
        case .cast(let movieID):
            MovieCastView(viewModel: …)
        case .person(let id):
            // Another feature's screen: only the app can build it.
            Text(verbatim: "Person \(id)")
        }
    }
}
```
