# Composition

The app target is the composition root. It is the only module that imports every feature, so it builds every service, wires features together, owns navigation state, and builds every screen.

## Contents
- App target layout
- XApp.swift
- Environment keys
- ContentView
- XScreen: tab-root screens and pushed screens

## App target layout

```
MoviesApp/
├── MoviesApp.swift                 @main: services, Router, Scene
├── App/ContentView.swift           TabView over Router.selectedTab
├── Adapters/                       consumer-owned ports implemented over other features
├── Extensions/Environment/         one @Entry key per injected service
├── Navigation/                     Router, Tab, Destination, DestinationView
└── Screens/                        one XScreen per screen
```

## XApp.swift

Build every client, service and adapter in `init()`. Store them as `let` properties typed as the feature protocols, and inject them on the `Scene` so every window and every screen sees the same instances. Use no DI container and no singletons.

```swift
@main
struct MoviesApp: App {
    @State private var router = Router()
    private let imageSession: URLSession
    private let discoverService: any DiscoverServiceProtocol
    private let movieDetailService: any MovieDetailServiceProtocol
    private let watchlistService: any WatchlistServiceProtocol
    private let watchlistMembershipService: any WatchlistMembershipServiceProtocol

    init() {
        imageSession = URLSession(configuration: .default)

        do {
            let client = try TMDBClient(session: .shared)
            let watchlistService = try WatchlistService.persistent()

            discoverService = DiscoverService(client: client)
            movieDetailService = MovieDetailService(client: client)
            self.watchlistService = watchlistService
            watchlistMembershipService = WatchlistMembershipAdapter(watchlist: watchlistService)
        } catch {
            fatalError("Couldn't start the app: \(error)")
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(router)
                .asyncImageURLSession(imageSession)
        }
        .environment(\.discoverService, discoverService)
        .environment(\.movieDetailService, movieDetailService)
        .environment(\.watchlistService, watchlistService)
        .environment(\.watchlistMembershipService, watchlistMembershipService)
    }
}
```

Rules:
- Live or mock is the app's decision (configuration, launch arguments for UI tests). Features never decide.
- One service instance per protocol for the app's lifetime; views never create services.
- App-wide policies (URL sessions, caches, tint, color scheme) are applied here, not in features or DesignSystem.

## Environment keys

One file per key in `Extensions/Environment/`, named `EnvironmentValues+XServiceProtocol.swift`. The default is the feature's mock, so previews and screens work without setup.

```swift
import DiscoverCore
import SwiftUI

extension EnvironmentValues {
    @Entry var discoverService: any DiscoverServiceProtocol = .mock
}
```

Only app screens read these keys. Feature views never read services from the environment; they receive view models.

The app always injects real instances, so a default only matters in previews. `.mock` returns a fresh instance each time; a preview that needs a mock's state to persist injects one stored instance.

## ContentView

```swift
struct ContentView: View {
    @Environment(Router.self) private var router

    var body: some View {
        @Bindable var router = router
        TabView(selection: $router.selectedTab) {
            SwiftUI.Tab(.appTabDiscover, systemImage: "sparkles", value: .discover) {
                DiscoverScreen()
            }
            SwiftUI.Tab(.appTabSearch, systemImage: "magnifyingglass", value: .search, role: .search) {
                SearchScreen()
            }
        }
    }
}
```

`SwiftUI.Tab` is module-qualified because the app declares its own `Tab` enum.

## XScreen

An `XScreen` is a small app view that composes one screen: it reads services from the environment, builds the feature's view model, and passes it to the feature's public view. There are two kinds.

### Tab-root screen

Owns the tab's `NavigationStack`, bound to the Router, and registers every destination type that can appear in that stack, once, at its root.

```swift
/// The Discover tab: its navigation stack, the Discover feature at the root, and the screens it can push.
struct DiscoverScreen: View {
    @Environment(\.discoverService) private var discoverService
    @Environment(Router.self) private var router

    var body: some View {
        @Bindable var router = router
        NavigationStack(path: $router[.discover]) {
            DiscoverView(viewModel: DiscoverViewModel(service: discoverService))
                .navigationDestination(for: DiscoverDestination.self) { destination in
                    DestinationView(destination: Destination(destination))
                }
                .navigationDestination(for: MovieDetailDestination.self) { destination in
                    DestinationView(destination: Destination(destination))
                }
                .navigationDestination(for: PersonDetailDestination.self) { destination in
                    DestinationView(destination: Destination(destination))
                }
        }
    }
}
```

### Pushed screen

Takes only identifiers, never models, so the stack stays small, `Codable` and restorable. Owns no stack.

```swift
/// A movie, pushed onto whichever stack opened it.
struct MovieDetailScreen: View {
    @Environment(\.movieDetailService) private var movieDetailService
    @Environment(\.watchlistMembershipService) private var watchlistMembershipService
    private let movieID: MovieSummary.ID

    init(movieID: MovieSummary.ID) {
        self.movieID = movieID
    }

    var body: some View {
        MovieDetailView(
            viewModel: MovieDetailViewModel(
                movieID: movieID,
                service: movieDetailService
            ),
            watchlistMembership: WatchlistMembership(
                movieID: movieID,
                service: watchlistMembershipService
            )
        )
    }
}
```

Rules:
- One `XScreen` per screen, named after the feature view it shows.
- Building the view model inline is fine: the feature view stores it in `@State`, so only the first instance is kept. Keep view model initializers cheap (assignments only).
- Every screen has a preview that injects `.mock` services; tab-root screens also inject `Router()`.
