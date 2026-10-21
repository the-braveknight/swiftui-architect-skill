# Conventions

## Contents
- Naming
- Folders

## Naming

`X` is the feature name in UpperCamelCase (`MovieDetail`); `x` is lowerCamelCase (`movieDetail`). A feature's other screens use their own noun in the same slots (`MovieCastService`, `MovieCastViewModel`, `MovieCastView`, `MovieCastScreen`).

Names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/): they read as English at the call site, name things by role rather than type, and avoid abbreviations. Things and side-effect-free reads are nouns, changes are imperative verbs, and Booleans read as assertions. The reasoning for services is in [domain](domain.md#naming-services-and-their-methods).

| Kind | Name | Example |
|---|---|---|
| Feature package | the capability | `Discover`, `MovieDetail`, `Watchlist` |
| Feature targets | `XCore`, `XUI`, `XLive`, `XTests` | `MovieDetailUI` |
| Shared packages | `Core` (with `CoreUI`), `DesignSystem`, `Networking` | |
| Domain model | a plain noun for the whole thing; never ends in `Detail` or a transport word | `Movie`, `Person`, `HomeSection` |
| What a list shows of a model | the model's name + `Summary` | `MovieSummary`, `HomeSectionSummary` |
| Feature error | `XError`, cased by what screens show; one per screen when their failures differ | `MovieDetailError.notFound`, `GenreDetailError` |
| Service protocol (XCore) | `XService`, one per screen | `DiscoverService`, `MovieCastService` |
| Service implementation (XLive) | its source + the protocol's name | `TMDBDiscoverService`, `SwiftDataWatchlistService` |
| Mock (XCore) | `Mock` + the protocol's name, exposed as `.mock`, and `.mock(_:)` for a scenario | `MockDiscoverService`, `.mock(.failure(.offline))` |
| Consumer-owned port | named for the consumer's need | `WatchlistMembershipService` |
| App adapter for a port | the port's role + `Adapter` | `WatchlistMembershipAdapter` |
| Service read | a noun phrase for what it returns | `movie(id:)`, `cast(movieID:)`, `sections(page:)` |
| Service change | an imperative verb phrase | `addToWatchlist(_:)`, `removeFromWatchlist(movieID:)` |
| A page from a service | `Page<Item>` (Core) with `items` and `nextPage` | `Page<HomeSectionSummary>` |
| API client and its errors | the API's name + `Client` / `Error`; client methods keep the spec's operation names | `TMDBClient.movieDetails(id:)`, `TMDBError` |
| View model | `XViewModel`, one per screen, and one per mode named after the views it drives | `MovieCastViewModel`, `GenreDetailSearchViewModel` |
| View model actions | imperative verbs | `load()`, `loadNextSectionsPage()`, `toggle(_:)` |
| View model state | per resource: the data named for what it is (`[]` for a collection, `nil` for a single value, until it loads), `isLoading…`, `…Error`; paged: `next…Page`, `isLoadingNext…Page`, `next…PageError`; keyed: the key (`selectedSeason`, `query`), and a plain `var` for input a view binds (`text`); computed `isLoading` and `error` only when a rule spans several resources | `shelves`, `isLoadingShelves`, `shelvesError`, `nextSectionsPage` |
| Focused model | `…ViewModel` when it only makes sense with the views it drives; otherwise a noun for what it is, no SwiftUI term of art | `WatchlistMembership` |
| Public screen view | `XView` | `MovieDetailView`, `MovieCastView` |
| State view | `XStateView`, picks the mode, then the content, the error, progress or the empty state | `DiscoverStateView` |
| Content view | `XContentView`, lays out the loaded values | `DiscoverContentView` |
| Mode views | the screen's names with the mode after `X` | `GenreDetailSearchStateView`, `GenreDetailSearchContentView`, `GenreDetailSearchNextPageFooter` |
| Sections | `XHeader`, `XRail`, `XRow`, `XCard`; one view per resource: `X` + resource + `View` | `MovieCastRail`, `HomeBannersView`, `HomeSectionsView` |
| Paging footer | `X` + resource + `NextPageFooter` | `HomeSectionsNextPageFooter` |
| Shared screen states | `ContentUnavailableView` presets in CoreUI, named for the situation | `.offline(retry:)`, `.unavailable(retry:)`, `.notFound`, `.noResults` |
| Toolbar | `XToolbar` (`ToolbarContent`) in XUI | `WatchlistToolbar` |
| Feature destinations | `XDestination`; cases name the place and carry IDs | `MovieDetailDestination.cast(movieID:)` |
| App navigation | `Router`, `TabDestination`, `Destination`, `DestinationView` | |
| App screen | `XScreen` | `MovieDetailScreen` |
| App `Destination` case | the app screen's name without `Screen` | `.movieDetail(id:)`, `.movieCast(movieID:)` |
| App entry point | `<AppName>App` | `MoviesApp` |
| Environment key | `xService` in `EnvironmentValues+XService.swift` | `\.discoverService` |
| Mapping file | `Model+Source.swift` | `Movie+TMDB.swift`, `MovieDetailError+TMDB.swift` |
| Display mapping | a property returning `LocalizedStringResource` | `DiscoverCategory.title` |
| Test fixture | `make…` | `makeMovie(id:title:)` |

## Folders

| Target | Folders |
|---|---|
| XCore | `Models/`, `Services/` |
| XUI | `Navigation/`, `Models/` (display mappings), `View Models/`, `Views/`, `Resources/` |
| XLive | `Mapping/`, `Services/` (and `Persistence/` when it stores data) |
| XTests | `ViewModels/`, `Live/`, `Fixtures.swift` |
| Core | `Domain/` |
| CoreUI | `Components/`, `Resources/` |
| Networking | `openapi.yaml`, `openapi-generator-config.yaml`, the client, its error, `Middleware/` |
| App | `App/`, `Adapters/`, `Extensions/Environment/`, `Navigation/`, `Screens/` |

One type per file, named after the type. Small private helper views may share their parent's file.
