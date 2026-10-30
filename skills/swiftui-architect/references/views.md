# Views

A screen is a small family of views in its feature's XUI target: `XView`, the public entry point, renders `XPhaseView`, which picks what to show from where the screen's loads stand, and `XContentView` lays the content out. Keep them plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique inside these shapes, use a SwiftUI skill ([related skills](../SKILL.md#related-skills)).

## Contents
- File roles
- XView, XPhaseView and XContentView
- What a screen shows without content
- Modes
- Passing data to views
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model and puts it in the environment; starts the screen's loads; applies what every state shares (title, search field, toolbars and the sheets they open); presents the screens the app passes in ([presenting a screen](navigation.md#presenting-a-screen)); renders `XPhaseView` |
| `XPhaseView` | Picks what to show from where the loads stand, as `AsyncImage` does from its phase: the mode when it's active, then the content, the error, progress or the empty state. It reads the view model's properties; there's no phase type |
| `XContentView` | Lays out the loaded values it's given in its scroll container, one view per resource or section |
| `XSearchPhaseView`, `XSearchContentView`, … | A [mode](#modes)'s own family, shaped like the screen's |
| `XHeader`, `XRail`, `XRow`, `XCard`, `X…View` | One section of this screen, taking only the values it shows |
| `Components/Cards/`, `Components/Rails/`, … | Views several of the feature's screens show, named for what they are ([inside a feature](modules.md#inside-a-feature)) |
| `X…NextPageFooter` | Loads the next page of a paged resource while it's on screen, offers a retry when that fails, and draws nothing after the last page |
| `XToolbar` | A `ToolbarContent` struct |
| `Model+Title.swift` | Display mappings from XCore values to `LocalizedStringResource` (`Department+Title.swift`) |

Where each file goes: [folders](conventions.md#folders). A feature exports one public view per screen (`MovieDetailView`, `MovieCastView`) and its view models; everything else is internal. One type per file.

## XView, XPhaseView and XContentView

```swift
// DiscoverView.swift
public struct DiscoverView: View {
    @State private var viewModel: DiscoverViewModel

    /// The view takes ownership of `viewModel` when it first appears; later values are ignored.
    public init(viewModel: DiscoverViewModel) {
        self.viewModel = viewModel
    }

    public var body: some View {
        DiscoverPhaseView()
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

// DiscoverPhaseView.swift
/// The shelves whenever there are some, with whatever featured movies have loaded; the error only when there are no
/// shelves, and progress while they load.
struct DiscoverPhaseView: View {
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
            LazyVStack(alignment: .leading) {
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
```

- `XView` never decides between states. It starts the screen's loads with `.task`, and `.task(id:)` for a [keyed resource](view-models.md#keyed-resources). Never start them on the progress view: it goes away when the first data arrives, and cancelling its task cancels the loads still running.
- What every state shares goes on `XView`, such as a control over the results, so it stays up while the screen loads, fails or shows content.
- `XPhaseView` checks in a fixed order: the content whenever there is data, the error only when there's nothing to show, progress while the load runs, then the empty state where the design has one. It follows the resource the content can't do without and passes the others along.
- A screen that loads nothing, such as a form, has no `XPhaseView`: `XView` renders `XContentView`.
- A screen with a [filter](view-models.md#filtering-loaded-values) shows the content while something matches, then `ContentUnavailableView.noResults` while there's data but nothing matches, ahead of the error and progress.
- `XContentView` takes values, owns the scroll container and lays out one view per resource or section. What acts inside it, such as the paging footer or a row's swipe action, reads the view model from the environment.
- A title from loaded data goes with that data. When the content is the thing the title names, the title goes in `XContentView` (`.navigationTitle(movie.title)`). When the thing loads beside the content, the title goes on `XView`, so it stays up in every state (`.navigationTitle(viewModel.genre?.name ?? "")`).

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

A [mode](view-models.md#modes), such as a search within a genre, replaces the screen's content while it's active. `XView` binds its field and runs its tasks beside the screen's, and `XPhaseView` picks it first:

```swift
// GenreDetailView.swift
public var body: some View {
    @Bindable var searchSession = viewModel.searchSession
    GenreDetailPhaseView()
        .navigationTitle(viewModel.genre?.name ?? "")
        .searchable(text: $searchSession.text, prompt: .discoverGenreDetailSearchPrompt)
        .onSubmit(of: .search) {
            searchSession.apply()
        }
        // Applies the words once typing pauses, and at once when the field is cleared.
        .task(id: searchSession.text) { … }
        .task(id: searchSession.query) {
            await searchSession.load()
        }
        // The frame, .task, .refreshable and .environment(viewModel), as in DiscoverView.
}

// GenreDetailPhaseView.swift
/// The search while there are words to search for; otherwise the genre's movies, the error or progress.
struct GenreDetailPhaseView: View {
    @Environment(GenreDetailViewModel.self) private var viewModel

    var body: some View {
        if viewModel.searchSession.isActive {
            GenreDetailSearchPhaseView()
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
- The phase view switches on `isActive`, which changes when words are applied ([typed text](view-models.md#keyed-resources)), not on each keystroke.
- The mode's family mirrors the screen's: a phase view checking its results in the same order (ending in `ContentUnavailableView.noResults`), a content view with its own `.refreshable`, and its own paging footer.
- The mode sits beside the content, never inside `XContentView`, which exists only once the screen's own data has loaded.

## Passing data to views

| Who | Holds | How |
|---|---|---|
| App `XScreen` | Creates the view model with every service it and its nested models use, and the screens `XView` presents | Inline in `body`, passed to `XView` |
| `XView` | Owns the view model for the screen's lifetime, binds its input, presents what its toolbar opens | `@State private var viewModel`, `.environment(viewModel)` outermost, `.sheet(isPresented: $viewModel.isFilterPresented)` |
| Views that decide from state or act: phase views, footers, toolbars | The view model, from the environment | `@Environment(GenreDetailViewModel.self) private var viewModel`; a nested model through it |
| Content and display views: sections, rows, cards | The values they show, never the view model | `DiscoverContentView(featured:shelves:)`, `movie: MovieSummary` |
| Focus and visual mechanics: a measured size, a scroll position, whether a row is on screen | The view itself | `@FocusState` or `@State private var` |

- Screen state lives in the view model, directly or in a nested model: what's presented, what a sheet edits, selections. `XView` holds no `@State` beside its view model, so a test can drive the whole screen through one type.
- Display views take values even though the view model is in reach, so the feature's other screens can show them too, and SharedUI can take one when a second feature needs it.
- Never pass closures down for the view model's actions: a view that acts reads the view model from the environment. A leaf whose one button fires one action, such as a preset's retry, may take a closure: pass a named private method (`.offline(retry: retry)`).
- A pushed screen never sees the previous screen's view model: each `XView` injects its own.

## View structure rules

- Choosing between states belongs in `XPhaseView`, never in a `Group { if … }`, `switch` or overlay in `XView`'s body.
- Never create views from non-view types: display mappings return `LocalizedStringResource` (`Department.title`), never `Text` or `Image`.
- Use native controls (`Button`, `NavigationLink`) with DesignSystem's styles, and strings through the catalog's generated symbols ([modules](modules.md#localization-assets-and-fonts)).

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
        .environment(SearchViewModel(service: .mock, recentSearchesService: .mock))
}
```

To click through a feature's own screens in previews, see [navigation previews](navigation.md#previews).
