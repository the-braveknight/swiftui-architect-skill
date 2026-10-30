# Conventions

## Contents
- Naming
- Folders

## Naming

`X` is a feature's name in UpperCamelCase (`Movie`), or one of its screens' (`MovieCast`); `x` is lowerCamelCase. A screen uses its noun in every slot (`MovieCastService`, `MovieCastViewModel`, `MovieCastView`, `MovieCastScreen`), and what spans the feature uses the feature's name (`MovieDestination`, `MovieError`). Which name a screen, model or shared view takes inside a feature is in [inside a feature](modules.md#inside-a-feature).

Names follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/): they read as English at the call site, name things by role, and avoid abbreviations. Things and side-effect-free reads are nouns, changes are imperative verbs, and Booleans read as assertions.

| Kind | Name | Example |
|---|---|---|
| Feature package | the capability, never a tab or a screen | `Discover`, `Movie`, `Watchlist`; not `MovieDetail` or `Home` |
| Feature targets | `XCore`, `XUI`, `XLive`, `XTests` | `MovieUI` |
| Domain model | a plain noun for the whole thing; never ends in `Detail` or a transport word | `Movie`, `Genre`, `DiscoverShelf` |
| What a list shows of a model | the model's name + `Summary` | `MovieSummary` |
| A model's id | `let id` of its real type, which `Identifiable` names `X.ID` | `Movie.ID` |
| A name an imported module declares | a qualifier from the domain | `MovieCollection`, not `Collection` |
| Feature error | `XError`; one per screen when their failures differ | `MovieError.notFound`, `GenreDetailError` |
| Service protocol (XCore) | `XService`, one per screen; another for each concern with a source of its own; never a `…Repository`, `…UseCase` or `…Manager` | `DiscoverService`, `MovieCastService`, `RecentSearchesService` |
| Service implementation (XLive) | its source + the protocol's name | `TMDBDiscoverService`, `TMDBWatchlistService` |
| Mock (XCore) | `Mock` + the protocol's name, exposed as `.mock`, and `.mock(_:)` for a scenario | `MockDiscoverService`, `.mock(.failure(.offline))` |
| Consumer-owned port, and its app adapter | named for the consumer's need; an adapter is the protocol's name with `Adapter` for `Service` | `WatchlistMembershipService`, `WatchlistMembershipAdapter` |
| Service reads and changes | noun phrases for what reads return; imperative verbs for changes | `movie(id:)`, `shelves(page:)`; `addToWatchlist(_:)` |
| API client and its error | the API's name + `Client` / `Error`; client methods keep the spec's operation names | `TMDBClient.movieDetails(id:)`, `TMDBError` |
| View model | `XViewModel`, one per screen | `MovieCastViewModel` |
| View model actions | imperative verbs; `refresh()` beside `load()` on a screen with a paged list | `load()`, `refresh()`, `loadNextShelvesPage()`, `select(_:)` |
| View model state | per resource: the data, `isLoading…`, `…Error`; paged: `next…Page`, `isLoadingNext…Page`, `next…PageError`; keyed: the key (`selectedDepartment`); bound input (`text`) and the values matching it (`matchingItems`), what's presented (`isFilterPresented`), a place shown over the screen (`presentedDestination`), the value a presentation is about (`itemToRemove`), one of several sheets (`presentedSheet`), a draft (`filterDraft`), a change in progress (`isUpdating`) ([view models](view-models.md#one-resource-data-activity-and-error)) | `shelves`, `isLoadingShelves`, `shelvesError`, `nextShelvesPage` |
| Nested model | declared inside its view model; a plain noun for what it holds, never `…ViewModel` | `MovieDetailViewModel.WatchlistMembership`, `GenreDetailViewModel.SearchSession` |
| Screen views | `XView` (public), `XPhaseView`, `XContentView`; a mode's after `X` | `DiscoverView`, `DiscoverPhaseView`, `GenreDetailSearchPhaseView` |
| Sections of one screen | `X` + what it is | `MovieDetailCastRail`, `DiscoverFeaturedView` |
| Views several of a feature's screens show | the model they show and their form, filed in `Components/` by kind | `CastMemberCard`, `MovieSummaryGrid` |
| Paging footer | `X` + resource + `NextPageFooter`, leaving out the resource when `X` already names it | `DiscoverShelvesNextPageFooter` |
| Toolbar | `XToolbar` (`ToolbarContent`) | `WatchlistToolbar` |
| Destinations | `XDestination` in a feature, `SharedDestination` in SharedUI; cases name the place and carry IDs | `MovieDestination.cast(movieID:)`, `SharedDestination.movieDetail(id:)` |
| App navigation | `Router`, `TabDestination`, `Destination`, `DestinationView`; a `Destination` case is its app screen's name without `Screen` | `.movieCast(movieID:)` |
| App screen | `XScreen` | `MovieDetailScreen` |
| App entry point | `<AppName>App` | `MoviesApp` |
| Environment key | `xService` in `EnvironmentValues+XService.swift` | `\.discoverService` |
| Mapping file | `Model+Source.swift` | `Movie+TMDB.swift`, `MovieError+TMDB.swift` |
| Display mapping | a property returning `LocalizedStringResource`, in `Model+Title.swift` | `Department.title` |
| Test fixture | `make…` | `makeMovie(id:title:)` |

## Folders

| Target | Folders |
|---|---|
| XCore | `Models/`; `Services/`, with a folder per service protocol holding it and its mock (`Services/MovieDetailService/`) |
| XUI | a folder per screen: its view model and nested models, views, toolbar, paging footers and display mappings, with a subfolder per mode or set of sections (`GenreDetail/Search/`, `MovieDetail/Sections/`); `Components/`: the views and display mappings several screens use, filed by kind (`Cards/`, `Rails/`); `Navigation/`; `Resources/` |
| XLive | `Mapping/`, `Services/` |
| XTests | `ViewModels/`, `Live/`, `Fixtures.swift` |
| Shared | `Domain/`; SharedUI: `Components/`, `Navigation/`, `Resources/`; SharedLive: `Mapping/`, and `Services/` for client steps several features share |
| Networking | `openapi.yaml`, `openapi-generator-config.yaml`, the client, its error, `Middleware/` |
| App | `App/`, `Adapters/`, `Extensions/Environment/`, `Navigation/`, `Screens/` |

One type per file, named after the type; a nested model's file adds it to its view model's name (`MovieDetailViewModel+WatchlistMembership.swift`). Small private helper views may share their parent's file.
