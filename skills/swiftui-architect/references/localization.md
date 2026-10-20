# Localization and assets

Every user-facing string comes from a String Catalog in the module that shows it, through symbols Xcode generates. Every asset comes from an asset catalog through generated symbols. No string literals and no string-named resources in UI code.

## Contents
- One catalog per module
- Keys and entries
- Using generated symbols
- Runtime values and interpolation
- Formatting and right-to-left
- Assets
- Build setup

## One catalog per module

| Module | Catalog | Holds |
|---|---|---|
| App | `MoviesApp/Localizable.xcstrings` | Tab titles, app-level screens (Settings), app chrome |
| Each `XUI` | `Sources/XUI/Resources/Localizable.xcstrings` | Everything the feature's views show |
| `CoreUI` | `Sources/CoreUI/Resources/Localizable.xcstrings` | Copy every feature must say the same way: shared captions and the shared screen states (offline, not found, something went wrong) |
| `XCore`, `XLive`, `Core`, `DesignSystem` | none | They show no text of their own |

Generated symbols are `internal` and carry their own bundle, so a feature's strings are private to that feature and resolve from the right bundle without `bundle:` arguments. A string two features show is duplicated in each catalog; it's the same text today but owned by two features. The exception is copy that must stay identical everywhere, such as the shared screen states: it lives once in CoreUI, next to the presets that show it.

## Keys and entries

- Keys are dotted, lowercase, scoped by feature: `feature.area.element`, words joined with `_`.
  `discover.navigation.title`, `movie_detail.section.cast.see_all`, `search.error.offline.message`.
- App keys start with `app.`: `app.tab.discover`, `app.settings.language`.
- Every entry has a translator comment and a translation for every supported language.
- Add the key to the catalog first, then use its symbol. An entry, as stored in `Localizable.xcstrings`:

```json
"discover.action.retry" : {
  "comment" : "Button that retries loading Discover after a failure.",
  "extractionState" : "manual",
  "localizations" : {
    "ar" : { "stringUnit" : { "state" : "translated", "value" : "حاول مرة أخرى" } },
    "en" : { "stringUnit" : { "state" : "translated", "value" : "Try Again" } }
  }
}
```

## Using generated symbols

Xcode turns `movie_detail.section.cast.see_all` into `LocalizedStringResource.movieDetailSectionCastSeeAll`. Use dot notation, and pass the symbol straight to APIs that accept a `LocalizedStringResource`:

```swift
.navigationTitle(.discoverNavigationTitle)
Button(.discoverActionRetry, action: retry)
Label(.watchlistActionAdd, systemImage: "bookmark")
Section(.appSettingsLanguage) { … }
Picker(.appSettingsLanguage, selection: $language) { … }
Tab(.appTabDiscover, systemImage: "sparkles", value: .discover) { … }
Rail(.discoverRailTrendingTitle) { … }
```

Use `Text(.symbol)` only where a view is needed: view-builder content, styled text, custom labels.

Don't:
- Write `Text("Retry")`, `String(localized:)` with literals, or `bundle: #bundle` by hand.
- Declare your own `LocalizedStringResource` extension with a generated name; it collides with the generated one.
- Put display strings in models or view models; map XCore values in an XUI extension instead (below).

## Runtime values and interpolation

Map XCore values to display strings with an extension in the feature's XUI target, so XCore stays free of UI strings:

```swift
extension DiscoverCategory {
    var title: LocalizedStringResource {
        switch self {
        case .trending: .discoverRailTrendingTitle
        case .nowPlaying: .discoverRailNowPlayingTitle
        case .popular: .discoverRailPopularTitle
        case .topRated: .discoverRailTopRatedTitle
        }
    }
}
```

An entry whose value has a placeholder (`%@`, `%lld`) generates a function: `No movies match “%@”.` becomes `Text(.searchStateNoResultsMessage(query))`. Plurals are catalog variations on the same entry; the symbol takes the count.

Content that comes from a backend (titles, overviews) is already localized by the backend; send the app's language with every request from the networking layer, and show the value as plain `String`.

## Formatting and right-to-left

- Numbers, dates, durations and lists use format styles: `Text(year, format: .number.grouping(.never))`, `Text(date, format: .dateTime.year().month().day())`, `Duration.formatted(.units(allowed: [.hours, .minutes]))`.
- Layout uses `leading`/`trailing`, never `left`/`right`.
- SF Symbols that imply direction mirror automatically. A custom image that implies direction sets its Direction attribute in the asset catalog so it mirrors in right-to-left.
- Check right-to-left in previews with `.environment(\.locale, Locale(identifier: "ar"))` and `.environment(\.layoutDirection, .rightToLeft)`.

## Assets

Asset catalogs generate symbols for colors and images, scoped to their module like strings:

```swift
Image(.emptyWatchlist)
Image(.brandLogo)
Color(.accentHighlight)
.foregroundStyle(.accentHighlight)
```

- Each module owns its assets: feature artwork in its XUI target, app icon and launch assets in the app, shared visuals in DesignSystem.
- DesignSystem colors are exposed as semantic tokens (`DesignSystem.Colors.Text.primary`), whether they are defined in code or in an asset catalog.
- Never `Image("name")` or `Color("name")`.

## Build setup

App target build settings:

```
LOCALIZATION_PREFERS_STRING_CATALOGS = YES
STRING_CATALOG_GENERATE_SYMBOLS = YES
SWIFT_EMIT_LOC_STRINGS = YES
ASSETCATALOG_COMPILER_GENERATE_SWIFT_ASSET_SYMBOL_EXTENSIONS = YES
```

Each XUI target declares `defaultLocalization: "en"` on its package and `resources: [.process("Resources")]` on the target.

Xcode generates string and asset symbols for package targets when it builds them.
