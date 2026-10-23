# Conventions

## Contents
- Naming
- Folders

## Naming

`X` is the feature name in UpperCamelCase (`Movie`); `x` is lowerCamelCase (`movie`). A feature's screens use their own noun in the same slots (`MovieDetailService`, `MovieDetailViewModel`, `MovieDetailView`, `MovieDetailScreen`; `MovieCastService`, …): a tab's root screen its tab's name, any other screen what it shows. Views and models several of its screens share are named for what they are, and what spans the feature uses the feature's name (`MovieDestination`, `MovieError`) ([inside a feature](modules.md#inside-a-feature)).

Names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/): they read as English at the call site, name things by role rather than type, and avoid abbreviations. Things and side-effect-free reads are nouns, changes are imperative verbs, and Booleans read as assertions. The reasoning for services is in [domain](domain.md#naming-services-and-their-methods).

| Kind | Name | Example |
|---|---|---|
| Feature package | the capability, never a tab or one of its screens | `Discover`, `Movie`, `Watchlist`; not `MovieDetail` or `Home` |
| Feature targets | `XCore`, `XUI`, `XLive`, `XTests` | `MovieUI` |
| Shared packages | `Shared` (with `SharedUI` and `SharedLive`), `DesignSystem`, `Networking` | |
| Domain model | a plain noun for the whole thing; never ends in `Detail` or a transport word | `Movie`, `Person`, `HomeSection` |
| What a list shows of a model | the model's name + `Summary` | `MovieSummary`, `HomeSectionSummary` |
| A model's id | `let id` of its real type, which `Identifiable` names `X.ID`; `typealias ID` only for a type that doesn't store its id | `Movie.ID`, `HomeSection.ID` |
| A name an imported module declares | a qualifier from the domain | `MovieCollection`, not `Collection` |
| Feature error | `XError`, cased by what screens show; one per screen when their failures differ | `MovieError.notFound`, `GenreDetailError` |
| Service protocol (XCore) | `XService`, one per screen; another for each concern with a source of its own | `DiscoverService`, `MovieCastService`, `RecentSearchesService` |
| Service implementation (XLive) | its source + the protocol's name | `TMDBDiscoverService`, `SwiftDataWatchlistService`, `UserDefaultsRecentSearchesService` |
| Mock (XCore) | `Mock` + the protocol's name, exposed as `.mock`, and `.mock(_:)` for a scenario | `MockDiscoverService`, `.mock(.failure(.offline))` |
| Consumer-owned port | named for the consumer's need | `WatchlistMembershipService` |
| App adapter for a port | the port's role + `Adapter` | `WatchlistMembershipAdapter` |
| Service read | a noun phrase for what it returns | `movie(id:)`, `cast(movieID:)`, `sections(page:)` |
| Service change | an imperative verb phrase | `addToWatchlist(_:)`, `removeFromWatchlist(movieID:)` |
| A page from a service | `Page<Item>` (Shared) with `items` and `nextPage` | `Page<HomeSectionSummary>` |
| API client and its errors | the API's name + `Client` / `Error`; client methods keep the spec's operation names | `TMDBClient.movieDetails(id:)`, `TMDBError` |
| View model | `XViewModel`, one per screen: pushed onto a stack, or a tab's root | `MovieCastViewModel` |
| View model actions | imperative verbs; `refresh()` beside `load()` on a screen with a paged list | `load()`, `refresh()`, `loadNextSectionsPage()`, `toggle(_:)` |
| View model state | per resource: the data named for what it is (`[]` for a collection, `nil` for a single value, until it loads), `isLoading…`, `…Error`; paged: `next…Page`, `isLoadingNext…Page`, `next…PageError`; keyed: the key (`selectedSeason`, `query`), and a plain `var` for input a view binds (`text`); presented: a plain `var` (`isFilterPresented`); edited in a sheet before it's applied: a plain `var` ending in `Draft` (`filterDraft`), filled by the action that opens the sheet (`openFilter()`); computed `isLoading` and `error` only when a rule spans several resources | `shelves`, `isLoadingShelves`, `shelvesError`, `nextSectionsPage` |
| Nested model | declared inside its view model; a plain noun for what it holds ([naming](view-models.md#naming-nested-models)), never `…ViewModel` | `GenreDetailViewModel.SearchSession`, `MovieDetailViewModel.WatchlistMembership` |
| Public screen view | `XView`; a tab's root takes its tab's name, any other screen what it shows | `DiscoverView`, `MovieDetailView`, `GenreDetailView` |
| State view | `XStateView`, picks the mode, then the content, the error, progress or the empty state | `DiscoverStateView` |
| Content view | `XContentView`, lays out the loaded values | `DiscoverContentView` |
| Mode views | the screen's names with the mode after `X` | `GenreDetailSearchStateView`, `GenreDetailSearchContentView`, `GenreDetailSearchNextPageFooter` |
| Sections | `XHeader`, `XRail`, `XRow`, `XCard`; one view per resource: `X` + resource + `View` | `MovieCastRail`, `HomeBannersView`, `HomeSectionsView` |
| Views several of a feature's screens show | what they are, without a screen's or a tab's name, filed in `Views/` by kind | `CastMemberCard`, `PosterRail`, `HeroBanner` |
| Paging footer | `X` + resource + `NextPageFooter` | `HomeSectionsNextPageFooter` |
| Shared screen states | `ContentUnavailableView` presets in SharedUI, named for the situation | `.offline(retry:)`, `.unavailable(retry:)`, `.notFound`, `.noResults` |
| Toolbar | `XToolbar` (`ToolbarContent`) in XUI | `WatchlistToolbar` |
| Feature geometry | `XLayout`, an enum in XUI the feature's views share; DesignSystem holds only domain-free tokens | `DiscoverLayout` |
| Feature destinations | `XDestination`; cases name the place and carry IDs | `MovieDestination.cast(movieID:)` |
| Shared destinations | `SharedDestination` in SharedUI: the places several features open, each case named for the screen it opens | `SharedDestination.movieDetail(id:)` |
| App navigation | `Router`, `TabDestination`, `Destination`, `DestinationView` | |
| App screen | `XScreen` | `MovieDetailScreen` |
| App `Destination` case | the app screen's name without `Screen` | `.movieDetail(id:)`, `.movieCast(movieID:)` |
| App entry point | `<AppName>App` | `MoviesApp` |
| Environment key | `xService` in `EnvironmentValues+XService.swift` | `\.discoverService` |
| Mapping file | `Model+Source.swift` | `Movie+TMDB.swift`, `MovieError+TMDB.swift` |
| Display mapping | a property returning `LocalizedStringResource` | `DiscoverCategory.title` |
| Test fixture | `make…` | `makeMovie(id:title:)` |

## Folders

| Target | Folders |
|---|---|
| XCore | `Models/`, `Services/` |
| XUI | `Navigation/`, `Models/` (display mappings), `View Models/`, `Views/` (a folder per screen, and one per kind of view several screens share: `Cards/`, `Rails/`), `Resources/` |
| XLive | `Mapping/`, `Services/` (and `Persistence/` when it stores data) |
| XTests | `ViewModels/`, `Live/`, `Fixtures.swift` |
| Shared | `Domain/` |
| SharedUI | `Components/`, `Navigation/`, `Resources/` |
| SharedLive | `Mapping/` |
| Networking | `openapi.yaml`, `openapi-generator-config.yaml`, the client, its error, `Middleware/` |
| App | `App/`, `Adapters/`, `Extensions/Environment/`, `Navigation/`, `Screens/` |

One type per file, named after the type; a nested model's file adds it to its view model's name (`MovieDetailViewModel+WatchlistMembership.swift`). Small private helper views may share their parent's file.
