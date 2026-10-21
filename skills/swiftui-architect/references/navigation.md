# Navigation

Native SwiftUI navigation only: `NavigationStack(path:)`, `NavigationLink(value:)`, `.navigationDestination(for:)`, plus local `sheet`, `fullScreenCover` and `alert`. The app owns navigation state and decides which screen each value shows; features only say where their views can go.

## Contents
- The model
- XDestination (feature)
- Pushing from feature views
- Router and TabDestination (app)
- Destination and DestinationView (app), and screens that don't exist yet
- Registering destinations
- Programmatic navigation
- Push or present
- Sheets, covers and alerts
- Previews

## The model

```
Feature view ── NavigationLink(value: MovieDetailDestination.cast(movieID: 550))
                      │  appended to the tab's NavigationPath
                      ▼
Tab-root screen ── .navigationDestination(for: MovieDetailDestination.self)
                      │  Destination(destination)  → .movieCast(movieID: 550)
                      ▼
DestinationView ── MovieCastScreen(movieID: 550) ── reads services, builds the view model
```

The same path works inside a feature (MovieDetail → its cast screen) and across features (MovieDetail → PersonDetail). The feature never knows which.

## XDestination (feature)

One enum per feature in `XUI/Navigation/XDestination.swift`: everywhere its views can take the user, its own screens and other features' screens alike, in the feature's own vocabulary.

```swift
/// Where MovieDetail's views can take the user. They push one with `NavigationLink(value:)`; the app
/// builds each screen and registers this type once, at the root of every stack that can show MovieDetail.
public enum MovieDetailDestination: Hashable, Codable, Sendable {
    /// A movie's details, e.g. a recommendation.
    case movie(id: MovieSummary.ID)
    /// Everyone credited in a movie.
    case cast(movieID: MovieSummary.ID)
    /// The person behind a cast credit.
    case person(id: Int)
}
```

- Cases carry identifiers (IDs, slugs, keys), never models or data to display. The pushed screen loads what it shows. `Codable` keeps stacks restorable and deep-linkable.
- Name cases for the place, not the action (`.person(id:)`, not `.showPerson`).

## Pushing from feature views

```swift
NavigationLink(value: MovieDetailDestination.movie(id: movie.id)) {
    MovieSummaryCard(movie: movie)
}
.buttonStyle(.content)

NavigationLink(.movieDetailSectionCastSeeAll, value: MovieDetailDestination.cast(movieID: movieID))
```

- In a `List`, hide the disclosure chevron when the design calls for it: `.navigationLinkIndicatorVisibility(.hidden)`.
- Feature views never receive navigation closures, a router, or a path.
- View models never navigate.

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

Inject it once from `XApp` with `.environment(router)`. The selected tab belongs in the Router, next to the stacks. Each tab-root screen binds its stack to its own entry (`NavigationStack(path: $router[.discover])`), and a tab that has never pushed gets an empty path.

`TabDestination` is the tabs' selection value, named like every other place the app can go (`XDestination`, `Destination`), which leaves SwiftUI's `Tab` free for `ContentView`. It stays a plain list of cases: each tab's title and icon are written in `ContentView`, with no extension mapping cases to them.

## Destination and DestinationView (app)

`Destination` lists the app's pushable screens. One initializer per feature is the only place a feature's destination becomes an app screen, so all cross-feature wiring is readable in one file. Several feature cases can open the same app screen: Discover's and MovieDetail's `.movie(id:)` both become `.movieDetail(id:)`.

```swift
/// The app's pushed screens. Features never see this type.
enum Destination: Hashable {
    case movieDetail(id: MovieSummary.ID)
    case movieCast(movieID: MovieSummary.ID)
    case personDetail(id: Person.ID)
}

extension Destination {
    init(_ destination: DiscoverDestination) {
        switch destination {
        case .movie(let id): self = .movieDetail(id: id)
        }
    }

    init(_ destination: MovieDetailDestination) {
        switch destination {
        case .movie(let id): self = .movieDetail(id: id)
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
        case .personDetail(let id):
            PersonDetailScreen(personID: id)
        }
    }
}
```

### Screens that don't exist yet

A feature links to a place in its own words even before that place is built, such as sign-in from a "Watch Now" button (`MovieDetailDestination.signIn`). The app maps it to a placeholder until the real screen ships:

```swift
case .signIn:
    ComingSoonScreen(.appDestinationSignIn)
```

When the screen arrives, only `DestinationView` changes; the features that link to it don't.

## Registering destinations

Every tab-root screen registers, at its stack root, each `XDestination` type that can appear in its stack, all in the same form:

```swift
.navigationDestination(for: MovieDetailDestination.self) { destination in
    DestinationView(destination: Destination(destination))
}
```

- Register only at stack roots. A registration inside a feature screen runs again when that screen appears twice in one stack (a recommendation pushes another movie), and SwiftUI reports a duplicate `navigationDestination`.
- The repetition across tabs is deliberate: each stack shows everything it can push, with no hidden registration.
- A link whose type isn't registered does nothing at runtime and SwiftUI logs it. When a feature becomes reachable from a new stack, register its destination there.

## Programmatic navigation

Only the app navigates in code, by changing Router state. Deep links and notifications resolve to a feature destination and append it:

```swift
router.selectedTab = .discover
router[.discover].append(MovieDetailDestination.movie(id: 550))
```

Pop to root by assigning an empty `NavigationPath()`. Because every destination is `Codable`, `NavigationPath.CodableRepresentation` can persist and restore stacks; make `TabDestination` `Codable` too to restore the selected tab.

## Push or present

- **Push** (`NavigationLink(value:)`) when the screen loads its own data, needs services, other features can reach it, or it belongs in the back stack.
- **Present** (`sheet`, `fullScreenCover`, `alert`) when the content is self-contained and built from values the presenting view already has: a full biography, a poster viewer, a confirmation. A presented view takes plain values, or the presenting screen's view model when it changes that screen's state, such as a filter it edits; never a service.

## Sheets, covers and alerts

What a screen presents is state on its view model, a plain `var` its views bind, and never goes through the app. Any view can open it, an action can close it (applying a filter closes the filter), and tests can check it. What only the presented view edits, such as a filter before it's applied, is that view's `@State`. `XView` presents what any state can open, such as a filter from its toolbar; a sheet only the content opens is presented there. A presented view that needs a title or a close button owns an unmanaged `NavigationStack`:

```swift
// PersonDetailViewModel
/// Whether the full biography is showing.
var isFullBiographyPresented = false

// PersonBiographySection
@Bindable var viewModel = viewModel
Button(.personDetailBiographyShowMore) { viewModel.isFullBiographyPresented = true }
    .sheet(isPresented: $viewModel.isFullBiographyPresented) {
        PersonBiographySheet(name: name, biography: biography)
    }
```

`PersonBiographySheet` takes those plain values, wraps its content in a `NavigationStack` for its title, and closes with a `ToolbarContent` struct holding `Button(role: .close) { dismiss() }`.

## Previews

Feature previews wrap the view in a `NavigationStack`. To click through a feature's own screens in previews, register its destinations on the preview root with a private, preview-only builder that uses mocks:

```swift
#Preview {
    NavigationStack {
        MovieDetailPreviewScreen(destination: .movie(id: 1))
            .navigationDestination(for: MovieDetailDestination.self) { destination in
                MovieDetailPreviewScreen(destination: destination)
            }
    }
}

/// Builds MovieDetail's own screens on mock services for previews. The app builds the real ones.
private struct MovieDetailPreviewScreen: View {
    let destination: MovieDetailDestination

    var body: some View {
        switch destination {
        case .movie(let id):
            MovieDetailView(viewModel: …)   // built on .mock services
        case .cast(let movieID):
            MovieCastView(viewModel: …)
        case .person(let id):
            // Another feature's screen: only the app can build it.
            Text(verbatim: "Person \(id)")
        }
    }
}
```
