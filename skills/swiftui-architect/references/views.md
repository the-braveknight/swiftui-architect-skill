# Views

A screen is a small family of views in its feature's XUI target: `XView`, the public entry point, renders `XStateView`, which picks what to show, and `XContentView` lays the content out. Keep them plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique inside these shapes, use a SwiftUI skill ([related skills](../SKILL.md#related-skills)).

## Contents
- File roles
- XView, XStateView and XContentView
- What a screen shows without content
- Modes
- Passing data to views
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model and puts it in the environment; starts the screen's loads; applies what every state shares (title, search field, toolbars and the sheets they open); renders `XStateView` |
| `XStateView` | Picks what to show: the mode when it's active, then the content, the error, progress or the empty state |
| `XContentView` | Lays out the loaded values it's given in its scroll container, one view per resource or section |
| `XSearchStateView`, `XSearchContentView`, … | A [mode](#modes)'s own family, shaped like the screen's |
| `XHeader`, `XRail`, `XRow`, `XCard`, `X…View` | One section of this screen, taking only the values it shows |
| `Components/Cards/`, `Components/Rails/`, … | Views several of the feature's screens show, named for what they are ([inside a feature](modules.md#inside-a-feature)) |
| `X…NextPageFooter` | Loads the next page of a paged resource while it's on screen |
| `XToolbar` | A `ToolbarContent` struct |
| `Model+Title.swift` | Display mappings from XCore values to `LocalizedStringResource` (`Department+Title.swift`) |

Where each file goes: [folders](conventions.md#folders). A feature exports one public view per screen (`MovieDetailView`, `MovieCastView`) and its view models; everything else is internal. One type per file.

## XView, XStateView and XContentView

```swift
// DiscoverView.swift
public struct DiscoverView: View {
    @State private var viewModel: DiscoverViewModel

    /// The view takes ownership of `viewModel` when it first appears; later values are ignored.
    public init(viewModel: DiscoverViewModel) {
        self.viewModel = viewModel
    }

    public var body: some View {
        DiscoverStateView()
            .frame(maxWidth: .infinity, maxHeight: .infinity)
            .navigationTitle(.discoverNavigationTitle)
            .task {
                await viewModel.load()
            }
            .refreshable {
                await viewModel.refresh()
            }
            // Outermost, so toolbars and sheets read it too.
            .environment(viewModel)
    }
}

// DiscoverStateView.swift
/// The shelves whenever there are some, with whatever featured movies have loaded; the error only when there are no
/// shelves, and progress while they load.
struct DiscoverStateView: View {
    @Environment(DiscoverViewModel.self) private var viewModel

    var body: some View {
        if !viewModel.shelves.isEmpty {
            DiscoverContentView(featured: viewModel.featured, shelves: viewModel.shelves)
        } else if let error = viewModel.shelvesError {
            switch error {
            case .offline:
                ContentUnavailableView.offline(retry: retry)
            case .unavailable:
                ContentUnavailableView.unavailable(retry: retry)
            }
        } else if viewModel.isLoadingShelves {
            ProgressView()
        }
    }

    private func retry() {
        Task { await viewModel.load() }
    }
}

// DiscoverContentView.swift
/// The featured movies over the shelves, with the next page loading as the end comes into view.
struct DiscoverContentView: View {
    let featured: [MovieSummary]
    let shelves: [DiscoverShelf]

    var body: some View {
        ScrollView {
            LazyVStack(alignment: .leading, spacing: DesignSystem.Spacing.space7) {
                if !featured.isEmpty {
                    DiscoverFeaturedView(movies: featured)
                }
                ForEach(shelves) { shelf in
                    DiscoverShelfRail(shelf: shelf)
                }
                DiscoverShelvesNextPageFooter()
            }
        }
    }
}

// DiscoverShelvesNextPageFooter.swift
/// Loads the next page while the end of the shelves is on screen, or offers a retry after a failure. Draws nothing
/// once the last page has loaded.
struct DiscoverShelvesNextPageFooter: View {
    @Environment(DiscoverViewModel.self) private var viewModel

    var body: some View {
        if viewModel.nextShelvesPageError != nil, !viewModel.isLoadingNextShelvesPage {
            Button(.discoverActionRetry, action: retry)
                .buttonStyle(.filled)
        } else if let nextPage = viewModel.nextShelvesPage {
            // Shows progress and loads while it's on screen, asking again after each page in case it's still in view.
            DiscoverShelvesNextPageLoader(nextPage: nextPage)
        }
    }

    private func retry() {
        Task { await viewModel.loadNextShelvesPage() }
    }
}
```

- `XView` never decides between states. It starts the screen's loads with `.task`, and `.task(id:)` for a [keyed resource](view-models.md#keyed-resources). Never start them on the progress view: it goes away when the first data arrives, and cancelling its task cancels the loads still running.
- What every state shares goes on `XView`, such as a control over the results. Inside `XStateView` or a content view it would be rebuilt whenever the state changes.
- `XStateView` checks in a fixed order: the content whenever there is data, the error only when there's nothing to show, progress while the load runs, then the empty state where the design has one. While there's data it reads only the data, so a refresh doesn't redraw it. It follows the resource the content can't do without and passes the others along.
- `XContentView` takes values, owns the scroll container and lays out one view per resource or section. A view inside it that acts, such as the paging footer, reads the view model from the environment itself.
- A title from loaded data goes with that data: in `XContentView` when the content is the thing it names (`.navigationTitle(movie.title)`), in `XView` when the thing loads beside the content, so it stays up in every state (`.navigationTitle(viewModel.genre?.name ?? "")`).

## What a screen shows without content

| Situation | Shown with | Retry |
|---|---|---|
| The device is offline | `ContentUnavailableView.offline(retry:)` | yes |
| What was opened doesn't exist, such as one movie's 404 | `ContentUnavailableView.notFound` | no |
| Any other failure | `ContentUnavailableView.unavailable(retry:)` | yes |
| An empty answer where the design has an empty state, such as an empty watchlist | `ContentUnavailableView` with the feature's own copy | no |
| An empty answer several features show alike, such as no search results | A SharedUI preset (`ContentUnavailableView.noResults`) | no |

- The presets live in SharedUI with their copy, because every screen must say the same thing about the same failure. A feature has no error view of its own.
- Where the design has no empty state, an empty answer shows nothing.
- A secondary resource never takes over the screen. A failed next page shows a short inline retry; a resource the screen can do without, such as the featured movies, is left out; a [keyed section](view-models.md#keyed-resources) shows its own progress and error.

## Modes

A [mode](view-models.md#modes), such as a search within a genre, replaces the screen's content while it's active. `XView` binds its field and runs its tasks beside the screen's, and `XStateView` picks it first:

```swift
// GenreDetailView.swift
public var body: some View {
    @Bindable var searchSession = viewModel.searchSession
    GenreDetailStateView()
        .navigationTitle(viewModel.genre?.name ?? "")
        .searchable(text: $searchSession.text, prompt: .genreDetailSearchPrompt)
        .onSubmit(of: .search) {
            searchSession.apply()
        }
        // Searches once typing pauses, and shows the genre again as soon as the field is cleared.
        .task(id: searchSession.text) {
            if !searchSession.text.isEmpty {
                try? await Task.sleep(for: .milliseconds(400))
                guard !Task.isCancelled else {
                    return
                }
            }
            searchSession.apply()
        }
        .task(id: searchSession.query) {
            await searchSession.load()
        }
        // The frame, .task, .refreshable and .environment(viewModel), as in DiscoverView.
}

// GenreDetailStateView.swift
/// The search while there are words to search for; otherwise the genre's movies, the error or progress.
struct GenreDetailStateView: View {
    @Environment(GenreDetailViewModel.self) private var viewModel

    var body: some View {
        if viewModel.searchSession.isActive {
            GenreDetailSearchStateView()
        } else if !viewModel.movies.isEmpty {
            GenreDetailContentView(movies: viewModel.movies)
        } else if let error = viewModel.moviesError {
            switch error { … }
        } else if viewModel.isLoadingMovies {
            ProgressView()
        }
    }
}
```

- The search field lives on `XView`, so it stays up while the screen loads, fails or searches.
- The state view reads only `isActive`, which changes when words are applied, not on each keystroke, so typing redraws nothing below `XView`.
- The mode's family mirrors the screen's: a state view checking its results in the same order (ending in `ContentUnavailableView.noResults`), a content view with its own `.refreshable`, and its own paging footer.
- The mode sits beside the content, never inside `XContentView`, which exists only once the screen's own data has loaded.

## Passing data to views

| Who | Holds | How |
|---|---|---|
| App `XScreen` | Creates the view model with every service it and its nested models use | Inline in `body`, passed to `XView` |
| `XView` | Owns the view model for the screen's lifetime, binds its input, presents what its toolbar opens | `@State private var viewModel`, `.environment(viewModel)` outermost, `.sheet(isPresented: $viewModel.isFilterPresented)` |
| Views that decide from state or act: state views, footers, toolbars | The view model, from the environment | `@Environment(GenreDetailViewModel.self) private var viewModel`; a nested model through it |
| Content and display views: sections, rows, cards | The values they show, never the view model | `DiscoverContentView(featured:shelves:)`, `movie: MovieSummary` |
| Focus and visual mechanics: a measured size, whether a footer is on screen, a carousel's page | The view itself | `@FocusState` or `@State private var` |

- Screen state lives in the view model, directly or in a nested model: what's presented, what a sheet edits, selections. `XView` holds no `@State` beside its view model, so a test can drive the whole screen through one type.
- Display views take values even though the view model is in reach, so the feature's other screens can show them too, and SharedUI can take one when a second feature needs it.
- Never pass closures down for the view model's actions. A view that takes a closure is re-evaluated whenever its parent is; one that reads the view model re-renders only for the properties it reads. A leaf whose one button fires one action, such as a preset's retry, may take a closure: pass a named private method (`.offline(retry: retry)`).
- A pushed screen never sees the previous screen's view model: each `XView` injects its own.

## View structure rules

- Choosing between states belongs in `XStateView`, never in a `Group { if … }`, `switch` or overlay in `XView`'s body.
- Never create views from non-view types: display mappings return `LocalizedStringResource` (`Department.title`), never `Text` or `Image`.
- Use native controls with DesignSystem's styles (`NavigationLink` with `.buttonStyle(.content)`), and strings through the catalog's generated symbols ([modules](modules.md#localization-assets-and-fonts)).

## Previews

Every public view has a preview built on mocks, so the feature is its own demo:

```swift
#Preview {
    NavigationStack {
        DiscoverView(viewModel: DiscoverViewModel(service: .mock))
    }
}

#Preview("Offline") {
    NavigationStack {
        DiscoverView(viewModel: DiscoverViewModel(service: .mock(.failure(.offline))))
    }
}

```

A view below `XView` reads a view model from the environment and crashes at runtime without one, so its preview injects the view model `XView` would have:

```swift
#Preview {
    Color.clear
        .sheet(isPresented: .constant(true)) {
            SearchFilterView()
        }
        .environment(SearchViewModel(service: .mock))
}
```

To click through a feature's own screens in previews, see [navigation previews](navigation.md#previews).
