# Views

A screen is a small family of views in its feature's XUI target: `XView`, the public entry point, renders `XStateView`, which picks what to show, and `XContentView` lays the content out. Keep them plain, native SwiftUI: a view model in, views out, `NavigationLink(value:)` for navigation. For SwiftUI technique beyond these shapes, use a SwiftUI skill.

## Contents
- File roles
- XView, XStateView and XContentView
- What a screen shows without content
- Several resources and the paging footer
- Passing data to views
- View structure rules
- Previews

## File roles

| File | Role |
|---|---|
| `XView` | Public entry point. Owns the view model as `@State`; applies screen modifiers; runs `load()` from `.task`; renders `XStateView` |
| `XStateView` | Shows the content, the error or progress; switches over the error and shows CoreUI's presets |
| `XContentView` | Lays out the content in its scroll container, one view per resource or section |
| `XHeader`, `XRail`, `XRow`, `XCard`, `XBannersView`, `XSectionsView` | One section, resource or element each, taking only the values it reads |
| `X…NextPageFooter` | Loads the next page of a paged resource while it's on screen; takes the view model |
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
        DiscoverStateView(viewModel: viewModel)
            .frame(maxWidth: .infinity, maxHeight: .infinity)
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

- `XView` never reads the data. It owns the view model, applies what every state shares, and starts `load()` from `.task`, so the load outlives the progress view ([cancellation](view-models.md#errors-and-cancellation)).
- `XStateView` only picks what to show: the content whenever there is some, the error only when there's nothing to show, and progress otherwise. It switches over the error itself and shows CoreUI's presets, so a feature has no error view of its own.
- `XContentView` owns the scroll container and lays out the content, one view per resource or section. With one resource it takes the data as a value (`DiscoverContentView(shelves:)`); with several it takes the view model, because a view such as a paging footer needs it.
- Anything that needs loaded data, such as a title from the model or a toolbar that only makes sense with content, is applied in `XContentView` (`.navigationTitle(movie.title)`).

## What a screen shows without content

| Situation | Shown with | Retry |
|---|---|---|
| The device is offline | `ContentUnavailableView.offline(retry:)` | yes |
| What was opened doesn't exist, such as a 404 for one item | `ContentUnavailableView.notFound` | no |
| Any other failure | `ContentUnavailableView.unavailable(retry:)` | yes |
| An empty answer where the design has an empty state, such as an empty watchlist or no search results | `ContentUnavailableView` with the feature's own copy | no |

- The presets are static members on `ContentUnavailableView` in CoreUI, with their copy, because every screen must say the same thing about the same failure. An empty state belongs to one feature, so it uses the feature's strings.
- An empty answer is never an error. Where the design has no empty state, the screen shows nothing.
- A secondary resource never takes over the screen. A failed next page or section shows a short inline message with a retry; one the screen can do without, such as suggestions, is simply left out.

## Several resources and the paging footer

The state view branches on the screen-wide `isLoading` and `error` ([view models](view-models.md#several-resources-computed-screen-state)). The content view lays out one view per resource; only the footer, which reads its own paging state and loads the next page, takes the view model:

```swift
// HomeStateView.swift
/// Renders the error or progress while Home has nothing to show, and its content otherwise.
struct HomeStateView: View {
    let viewModel: HomeViewModel

    var body: some View {
        if let error = viewModel.error {
            switch error {
            case .offline:
                ContentUnavailableView.offline(retry: retry)
            case .unavailable:
                ContentUnavailableView.unavailable(retry: retry)
            }
        } else if viewModel.isLoading {
            ProgressView()
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
            Button(.homeActionRetry, action: retry)
                .buttonStyle(.filled)
        } else if let nextPage = viewModel.nextSectionsPage {
            HomeSectionsNextPageLoader(nextPage: nextPage, viewModel: viewModel)
        }
    }

    private func retry() {
        Task { await viewModel.loadNextSectionsPage() }
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

## Passing data to views

| Who | Holds | How |
|---|---|---|
| App `XScreen` | Creates view models and focused models | Inline in `body`, passed to `XView` |
| `XView` | Owns them for the screen's lifetime | `@State private var viewModel` |
| State and content views, footers | Read what they need | Plain `let viewModel`; Observation tracks each property read |
| Display views (sections, rows, cards) | The values they show | `title: String`, `movie: MovieSummary`, `sections: [HomeSectionSummary]` |
| A view that needs a binding | Derives one | `@Bindable var viewModel = viewModel` in `body` |
| Presentation flags, form input, focus | The view that presents or edits | `@State private var` |

- A view takes the view model only when it reads state of its own, such as a resource's loading flag or error, or calls the view model, like a paging footer.
- Don't pass closures down through views for the view model's actions. A closure can't be compared, so a view that takes one is re-evaluated whenever its parent is, while a view holding the view model re-renders only for the properties it reads.
- A leaf whose button fires one action, such as a preset's retry, may take a closure, since nothing below it can be skipped. Pass a private method of the view that makes the call (`.offline(retry: retry)`), so the action has a name.

## View structure rules

- One `View` struct per section. No computed `some View` properties and no `@ViewBuilder` helper functions.
- No wrapper struct around a single system view, such as a label around `Label` or an icon view around one `Image`. Extract a struct only when it holds real layout or logic.
- Choosing between a screen's states belongs in `XStateView`, not in a `Group { if … }` or `switch` in `XView`'s body.
- Toolbars are `ToolbarContent` structs in XUI (`WatchlistToolbar`).
- Never create views from non-view types. View models expose domain values; display mappings are XUI extensions returning `LocalizedStringResource` (`DiscoverCategory.title`), never `Text` or `Image`.
- Use native controls with DesignSystem styles: `Button` with `.buttonStyle(.filledProminent)`, `NavigationLink` with `.buttonStyle(.content)`, `ProgressView()`.
- Strings come from the feature's String Catalog through generated symbols ([modules](modules.md#localization-and-assets)).

## Previews

Every public view has a preview built on mocks, so the feature is its own demo and never needs the app to be seen:

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

To click through a feature's own screens in previews, see [navigation previews](navigation.md#previews).
