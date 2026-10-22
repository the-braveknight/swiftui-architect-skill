# Views

A screen is a small family of views in its feature's XUI target: `XView`, the public entry point, renders `XStateView`, which picks what to show, and `XContentView` lays the content out. Keep them plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique inside these shapes (state, modifiers, layout, navigation APIs), use a SwiftUI skill ([related skills](../SKILL.md#related-skills)).

## Contents
- File roles
- XView, XStateView and XContentView
- What a screen shows without content
- Several resources and the paging footer
- Modes
- Passing data to views
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model as `@State` and puts it in the environment; applies what every state shares (title, search field, a bar of controls above every state, toolbars and the sheets they open); starts every load from `.task`, and `.task(id:)` for keyed resources; renders `XStateView` |
| `XStateView` | Reads the view model from the environment; picks the mode when it's active, then the content, the error, progress or the empty state; passes the data to the content view as values; switches over the error and shows CoreUI's presets |
| `XContentView` | Takes the loaded values and lays them out in its scroll container, one view per resource or section |
| `XSearchStateView`, `XSearchContentView`, `XSearchNextPageFooter` | A [mode](#modes)'s own family, shaped like the screen's and reading the mode through the view model |
| `XHeader`, `XRail`, `XRow`, `XCard`, `XBannersView`, `XSectionsView` | One section, resource or element each, taking only the values it reads |
| `X…NextPageFooter` | Loads the next page of a paged resource while it's on screen; reads the view model from the environment |
| `XToolbar` | A `ToolbarContent` struct; an item that opens a screen pushes an `XDestination` value |
| `Models/X+Title.swift` | Display mappings from XCore values to `LocalizedStringResource` |

A feature exports one public view per screen it owns (`MovieDetailView`, `MovieCastView`) and its view models. Everything else stays internal. Each of these is its own file.

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
                await viewModel.load()
            }
            // Outermost, so toolbars and sheets read it too.
            .environment(viewModel)
    }
}

// DiscoverStateView.swift
/// Renders the shelves whenever there are some, the error only when there's nothing to show, and progress while the
/// first load runs. A retry loads again while the error stays on screen.
struct DiscoverStateView: View {
    @Environment(DiscoverViewModel.self) private var viewModel

    var body: some View {
        if !viewModel.shelves.isEmpty {
            DiscoverContentView(shelves: viewModel.shelves)
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
        }
    }
}
```

- `XView` never decides between states. It owns the view model, puts it in the environment as its outermost modifier, applies what every state shares, and starts every load: `.task` for the screen's, so the load outlives the progress view ([cancellation](view-models.md#errors-and-cancellation)), and `.task(id:)` for a [keyed resource](view-models.md#keyed-resources). It's the only view that injects the view model ([passing data](#passing-data-to-views)).
- A control every state shares, such as filters over the results, goes on `XView` too. Inside `XStateView` or a content view it would be rebuilt whenever the state changes, such as while the results it filters reload.
- `XStateView` only picks what to show, in a fixed order: the content whenever there is data, the error only when there's nothing to show, progress while the load runs, and otherwise the empty state where the design has one. While there's data it reads only the data, so a refresh starting or ending doesn't redraw it. It switches over the error itself and shows CoreUI's presets, so a feature has no error view of its own.
- `XContentView` takes the values `XStateView` passes down (`DiscoverContentView(shelves:)`), owns the scroll container and lays them out, one view per resource or section. A view inside it that acts, such as a paging footer, reads the view model from the environment, so the content view doesn't pass it along; the content view reads it only to act itself, such as a mode's `.refreshable`.
- A title from loaded data goes with that data: in `XContentView` when the content is the thing it names (`.navigationTitle(movie.title)`), in `XView` when the thing loads beside the content, so it stays up in every state (`.navigationTitle(viewModel.genre?.name ?? "")`). A toolbar that only makes sense with content goes in `XContentView`.

## What a screen shows without content

| Situation | Shown with | Retry |
|---|---|---|
| The device is offline | `ContentUnavailableView.offline(retry:)` | yes |
| What was opened doesn't exist, such as a 404 for one item | `ContentUnavailableView.notFound` | no |
| Any other failure | `ContentUnavailableView.unavailable(retry:)` | yes |
| An empty answer where the design has an empty state, such as an empty watchlist | `ContentUnavailableView` with the feature's own copy | no |
| An empty answer the design shows the same way on several screens, such as no search results | A CoreUI preset (`ContentUnavailableView.noResults`) | no |

- The presets are static members on `ContentUnavailableView` in CoreUI, with their copy, because every screen must say the same thing about the same failure. An empty state that belongs to one feature uses the feature's strings; one that several features show alike is a preset too.
- An empty answer is never an error. Where the design has no empty state, the screen shows nothing.
- A secondary resource never takes over the screen. A failed next page or section shows a short inline message with a retry; one the screen can do without, such as suggestions, is simply left out.

## Several resources and the paging footer

The state view follows the resource the content can't do without ([view models](view-models.md#several-resources)) and passes the others' values along. The content view lays out one view per resource; only the footer, which reads its own paging state and loads the next page, reads the view model:

```swift
// HomeStateView.swift
/// Renders Home whenever there are sections, with whatever banners have loaded; the error only when there are no
/// sections, and progress while they load.
struct HomeStateView: View {
    @Environment(HomeViewModel.self) private var viewModel

    var body: some View {
        if !viewModel.sections.isEmpty {
            HomeContentView(banners: viewModel.banners, sections: viewModel.sections)
        } else if let error = viewModel.sectionsError {
            switch error {
            case .offline:
                ContentUnavailableView.offline(retry: retry)
            case .unavailable:
                ContentUnavailableView.unavailable(retry: retry)
            }
        } else if viewModel.isLoadingSections {
            ProgressView()
        }
    }

    private func retry() {
        Task { await viewModel.load() }
    }
}

// HomeContentView.swift
/// The banners above the sections, with the next page loading as the end comes into view.
struct HomeContentView: View {
    let banners: [MovieSummary]
    let sections: [HomeSectionSummary]

    var body: some View {
        ScrollView {
            VStack(spacing: DesignSystem.Spacing.space5) {
                if !banners.isEmpty {
                    HomeBannersView(banners: banners)
                }
                HomeSectionsView(sections: sections)
                HomeSectionsNextPageFooter()
            }
        }
    }
}

// HomeSectionsNextPageFooter.swift
/// Loads the next page while the end of the sections is on screen, or offers a retry after a failure. Draws nothing
/// once the last page has loaded.
struct HomeSectionsNextPageFooter: View {
    @Environment(HomeViewModel.self) private var viewModel

    var body: some View {
        if viewModel.nextSectionsPageError != nil, !viewModel.isLoadingNextSectionsPage {
            Button(.homeActionRetry, action: retry)
                .buttonStyle(.filled)
        } else if let nextPage = viewModel.nextSectionsPage {
            HomeSectionsNextPageLoader(nextPage: nextPage)
        }
    }

    private func retry() {
        Task { await viewModel.loadNextSectionsPage() }
    }
}

private struct HomeSectionsNextPageLoader: View {
    let nextPage: Int
    @Environment(HomeViewModel.self) private var viewModel
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

## Modes

A [mode](view-models.md#modes) replaces the screen's content while it's active, such as a server search within a genre. It's a nested model on the screen's view model, and `XStateView` picks it in its first branch:

```swift
// GenreDetailView.swift
public var body: some View {
    @Bindable var searchSession = viewModel.searchSession
    GenreDetailStateView()
        .frame(maxWidth: .infinity, maxHeight: .infinity)
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
        .task {
            await viewModel.load()
        }
        .refreshable {
            await viewModel.refresh()
        }
        .environment(viewModel)
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

// GenreDetailSearchStateView.swift
/// The results for the words, the error, progress or no results.
struct GenreDetailSearchStateView: View {
    @Environment(GenreDetailViewModel.self) private var viewModel

    var body: some View {
        if !viewModel.searchSession.results.isEmpty {
            GenreDetailSearchContentView(results: viewModel.searchSession.results)
        } else if let error = viewModel.searchSession.resultsError {
            switch error { … }
        } else if viewModel.searchSession.isLoadingResults {
            ProgressView()
        } else {
            ContentUnavailableView.noResults
        }
    }
}
```

- `XView` holds what every state shares, the search field included, so the field stays up while the screen loads, fails or searches. It binds the field to the mode's text and runs the mode's tasks beside the screen's.
- The state view reads only `isActive` from the mode, which changes when words are applied, not on each keystroke. Typing redraws `XView`, whose children's inputs haven't changed, and nothing below it.
- The mode's family is shaped like the screen's: a state view that checks its results in the same order, a content view with its own scroll container and `.refreshable`, which refreshes the results rather than the screen, and its own paging footer. They read the mode through the screen's view model (`viewModel.searchSession`).
- The mode sits beside the content, not inside `XContentView`. The content view exists only once the screen's own data has loaded, so a mode inside it couldn't show while that data loads or after it fails, and the content view would end up deciding states.

## Passing data to views

| Who | Holds | How |
|---|---|---|
| App `XScreen` | Creates the screen's view model with every service it and its nested models use | Inline in `body`, passed to `XView` |
| `XView` | Owns it for the screen's lifetime, puts it in the environment, binds its input and presents what its toolbar opens | `@State private var viewModel`; `.environment(viewModel)` as its outermost modifier; `@Bindable var searchSession = viewModel.searchSession` in `body`; `.sheet(isPresented: $viewModel.isFilterPresented)` |
| State views, footers, toolbars, and sections that read their own state or act | Read the data, errors, loading and paging state they decide from, and call actions | `@Environment(GenreDetailViewModel.self) private var viewModel`, and a nested model through it (`viewModel.searchSession`); Observation tracks each property read |
| Content views | The loaded values | `GenreDetailContentView(movies:)`; the environment only when the content view acts itself, such as a mode's `.refreshable` |
| Display views (sections, rows, cards) | The values they show, never the view model | `title: String`, `movie: MovieSummary`, `sections: [HomeSectionSummary]` |
| A view that needs a binding | Derives one | `@Bindable var viewModel = viewModel` in `body` |
| What a screen presents | Its view model | A plain `var` the view binds (`isFilterPresented`) |
| What a sheet edits before it's applied | Its view model | A plain `var` (`filterDraft`) that the action opening the sheet fills from the screen's state (`openFilter()`); the sheet binds it, so `SearchFilterView()` takes nothing ([presentations](navigation.md#sheets-covers-and-alerts)) |
| Focus and visual mechanics: a measured size, whether a footer is on screen, gesture progress | The view | `@FocusState` or `@State private var`; never data, a presentation or an edit a test would check |

- `XView` holds no `@State` beside its view model, and no view below it holds screen state. Anything a test, a deep link or another view could need lives in the view model, directly or in a nested model, so a screen's whole state is one type a test can drive.
- A view reads the view model from the environment only when it reads state of its own, such as a resource's loading flag or error, or calls the view model, like a paging footer. A view that only passes data along takes the values instead.
- Display views take values even though the view model is in reach, so a row never depends on one screen and can move to CoreUI when a second feature needs it.
- `XView` injects once per screen, as its outermost modifier, so its toolbar items and the sheets it presents read the view model too. Nested models aren't injected separately. A pushed screen never sees the previous screen's view model, since destinations are registered at the stack root and each `XView` injects its own.
- A view reading a view model that wasn't injected crashes at runtime rather than failing to build, so a preview of any view below `XView` injects one ([previews](#previews)).
- Don't pass closures down through views for the view model's actions. A closure can't be compared, so a view that takes one is re-evaluated whenever its parent is, while a view reading the view model re-renders only for the properties it reads.
- A leaf whose button fires one action, such as a preset's retry, may take a closure, since nothing below it can be skipped. Pass a private method of the view that makes the call (`.offline(retry: retry)`), so the action has a name.

## View structure rules

- One `View` struct per section. No computed `some View` properties and no `@ViewBuilder` helper functions.
- No wrapper struct around a single system view, such as a label around `Label` or an icon view around one `Image`. Extract a struct only when it holds real layout or logic.
- Choosing between a screen's states, a mode included, belongs in `XStateView`, not in a `Group { if … }`, `switch` or overlay in `XView`'s body.
- Toolbars are `ToolbarContent` structs in XUI (`WatchlistToolbar`).
- Never create views from non-view types. View models expose domain values; display mappings are XUI extensions returning `LocalizedStringResource` (`DiscoverCategory.title`), never `Text` or `Image`.
- Use native controls with DesignSystem styles: `Button` with `.buttonStyle(.filledProminent)`, `NavigationLink` with `.buttonStyle(.content)`, `ProgressView()`.
- Strings come from the feature's String Catalog through generated symbols ([modules](modules.md#localization-assets-and-fonts)).

## Previews

Every public view has a preview built on mocks, so the feature is its own demo and never needs the app to be seen:

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

#Preview("Arabic") {
    NavigationStack {
        DiscoverView(viewModel: DiscoverViewModel(service: .mock))
    }
    .environment(\.locale, Locale(identifier: "ar"))
    .environment(\.layoutDirection, .rightToLeft)
}
```

A mock that takes a scenario shows each state the screen has, its errors included ([mocks](domain.md#mocks)).

A preview of a view below `XView`, such as a sheet, injects the view model `XView` would have:

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
