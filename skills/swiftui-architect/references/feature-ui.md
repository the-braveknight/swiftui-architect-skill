# Feature UI

What lives in an `XUI` target and the shape of each piece. Keep feature views plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique beyond these shapes, use a SwiftUI skill.

## Contents
- File roles
- XView and XContentView
- View models
- Focused models
- Ownership
- LoadState and PagedCollection
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model as `@State`; applies screen modifiers; renders `XContentView` |
| `XContentView` | Switches once over `viewModel.state` |
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

/// Renders Discover's current state. Reads only `state`, so it re-renders only when the state changes.
struct DiscoverContentView: View {
    let viewModel: DiscoverViewModel

    var body: some View {
        switch viewModel.state {
        case .idle, .loading:
            ProgressView()
                .progressViewStyle(.loadingDots)
        case .loaded(let shelves):
            ScrollView {
                DiscoverRailList(shelves: shelves)
            }
        case .failed(let error):
            DiscoverErrorView(error: error) {
                Task { await viewModel.load() }
            }
        }
    }
}
```

Anything that needs loaded data, such as a title from the model or a toolbar that only makes sense with content, is applied inside the `.loaded` branch (`.navigationTitle(movie.title)`), so `XView` never reads `state`.

## View models

```swift
@MainActor
@Observable
public final class DiscoverViewModel {
    private let service: any DiscoverServiceProtocol

    private(set) var state: LoadState<[DiscoverShelf], DiscoverError> = .loading

    public init(service: any DiscoverServiceProtocol) {
        self.service = service
    }

    func load() async {
        if state.value == nil { state = .loading }
        do {
            state = .loaded(try await service.loadShelves())
        } catch is CancellationError {
            // The view went away; not a failure.
        } catch {
            // A failed refresh keeps the content already on screen.
            if state.value == nil { state = .failed(error as? DiscoverError ?? .unavailable) }
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

## LoadState and PagedCollection

Both live in CoreUI.

- `LoadState<Value, Failure>`: `.idle`, `.loading`, `.loaded(Value)`, `.failed(Failure)`. `Failure` is the feature's own error, so views switch over it exhaustively.
- Three rules every view model follows: cancellation is never a failure; a failed refresh keeps loaded content; errors are mapped into the feature's error before reaching state.
- `PagedCollection<Element, Failure>`: elements loaded page by page, with a `nextPage` state (`.available`, `.loading`, `.failed`, `.exhausted`). A failed first page is `LoadState.failed`; a failed later page is `nextPage == .failed` and keeps the loaded elements.

## View structure rules

- Separate `View` structs for every section. No computed `some View` properties and no `@ViewBuilder` helper functions.
- No `Group { switch … }` in a body: move the switch into an `XContentView` and apply modifiers outside it.
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
