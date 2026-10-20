# Conventions

## Contents
- Naming
- Folders
- Formatting

## Naming

`X` is the feature name in UpperCamelCase (`MovieDetail`); `x` is lowerCamelCase (`movieDetail`). A feature's other screens use their own noun in the same slots (`MovieCastView`, `MovieCastViewModel`, `MovieCastScreen`).

| Kind | Name | Example |
|---|---|---|
| Feature package | the capability | `Discover`, `MovieDetail`, `Watchlist` |
| Feature targets | `XCore`, `XUI`, `XLive`, `XTests` | `MovieDetailUI` |
| Shared package and targets | `Core`, `CoreUI` | |
| Domain model | a plain noun; never ends in `Detail` | `Movie`, `Person`, `CastMember` |
| Model shared across features | names what it is in lists | `MovieSummary` |
| Feature error | `XError` | `MovieDetailError` |
| Service protocol (XCore) | `XServiceProtocol` | `DiscoverServiceProtocol` |
| Live service (XLive) | `XService` | `DiscoverService` |
| Mock (XCore) | `MockXService`, exposed as `.mock` | `MockDiscoverService` |
| Consumer-owned port | named for the consumer's need | `WatchlistMembershipServiceProtocol` |
| App adapter for a port | port name + `Adapter` | `WatchlistMembershipAdapter` |
| Service method | the value it returns | `movie(id:)`, `cast(movieID:)`, `loadShelves()`, `loadSections(page:)` |
| View model | `XViewModel`, one per screen | `MovieCastViewModel` |
| View model state | per resource: the data named for what it is, `isLoading…`, `…Error`; paged: `next…Page`, `isLoadingNext…Page`, `next…PageError`; computed `isLoading` and `error` only on screens with several resources. Each error property has its own type | `shelves`, `isLoadingShelves`, `shelvesError`, `nextSectionsPage` |
| A page from a service | `Page<Item>` (Core) with `items` and `nextPage` | `Page<HomeSection>` |
| Focused model | a noun for what it is, no `ViewModel` suffix, no SwiftUI term of art | `WatchlistMembership` |
| Public screen view | `XView` | `MovieDetailView`, `MovieCastView` |
| State view | `XStateView`, picks the content, the error or progress | `DiscoverStateView` |
| Content view | `XContentView`, lays out the content | `DiscoverContentView` |
| Sections | `XHeader`, `XRail`, `XRow`, `XCard`; one view per resource: `X` + resource + `View` | `MovieCastRail`, `HomeBannersView`, `HomeSectionsView` |
| Paging footer | `X` + resource + `NextPageFooter` | `HomeSectionsNextPageFooter` |
| Shared screen states | `ContentUnavailableView` presets in CoreUI, named for the situation | `.offline(retry:)`, `.unavailable(retry:)`, `.notFound` |
| Toolbar | `XToolbar` (`ToolbarContent`) in XUI | `WatchlistToolbar` |
| Feature destinations | `XDestination` | `MovieDetailDestination` |
| App navigation | `Router`, `TabDestination`, `Destination`, `DestinationView` | |
| App screen | `XScreen` | `MovieDetailScreen` |
| App `Destination` case | the app screen's name without `Screen` | `.movieDetail(id:)`, `.movieCast(movieID:)` |
| App entry point | `<AppName>App` | `MoviesApp` |
| Environment key | `xService` in `EnvironmentValues+XServiceProtocol.swift` | `\.discoverService` |
| DTO mapping file | `Model+Source.swift` | `Movie+TMDB.swift` |
| Display mapping | a property returning `LocalizedStringResource` | `DiscoverCategory.title` |
| Style | its look + `Style`, with a static member for its role | `ChipButtonStyle` → `.chip`, `LoadingDotsStyle` → `.loadingDots` |
| String key | `feature.area.element` → symbol `.featureAreaElement` | `discover.action.retry` → `.discoverActionRetry` |
| Test fixture | `make…` | `makeMovie(id:title:)` |

Methods and properties follow the Swift API Design Guidelines: read as English at the call site, name by role not type, no abbreviations.

## Folders

| Target | Folders |
|---|---|
| XCore | `Models/`, `Services/` |
| XUI | `Navigation/`, `Models/` (display mappings), `View Models/`, `Views/`, `Resources/` |
| XLive | `Mapping/`, `Services/` (and `Persistence/` when it stores data) |
| XTests | `ViewModels/`, `Live/`, `Fixtures.swift` |
| Core | `Domain/` |
| CoreUI | `Components/` |
| App | `App/`, `Adapters/`, `Extensions/Environment/`, `Navigation/`, `Screens/` |

One type per file, named after the type. Small private helper views may share their parent's file.

## Formatting

Format with `swift-format` and a repository `.swift-format`: the defaults plus 4-space indentation, 120-column lines, `lineBreakBeforeEachArgument: true`, `respectsExistingLineBreaks: true`. After editing:

```bash
find App Packages -name '*.swift' -not -path '*/.build/*' -print0 \
  | xargs -0 swift-format format --in-place --parallel --configuration .swift-format
swift-format lint --strict -r App Packages
```

Beyond the formatter:
- When a call constructs objects inline, a nested initializer with two or more arguments puts one argument per line, at every level, even if the call fits on one line:

```swift
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
```

- A single-argument nesting stays on one line: `DiscoverView(viewModel: DiscoverViewModel(service: discoverService))`.
- View-builder and registration closures name their parameter and put the body on its own line; `$0` is fine in one-line transforms (`map`, `filter`):

```swift
.navigationDestination(for: MovieDetailDestination.self) { destination in
    DestinationView(destination: Destination(destination))
}
```

- Don't pass one closure inside the parentheses and another as a trailing closure in the same call. Use trailing closures for all of them (`Rail(.title) { … } accessory: { … }`) or label every one.
- Doc comments (`///`) explain why a type or decision exists, not what the code already says.
