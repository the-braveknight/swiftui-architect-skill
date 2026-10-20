# Design system

`DesignSystem` is one domain-free package that every UI target imports. It holds the visual language: tokens, styles for native controls, and generic components. It imports SwiftUI only and never knows a feature model, service, destination or screen.

## Contents
- Layout
- Foundations (tokens)
- Styles: XStyle conformances
- Components
- Rules

## Layout

```
Packages/DesignSystem/Sources/DesignSystem/
├── Foundations/   DesignSystem.swift (namespace), SemanticColors, ColorPrimitives, Typography,
│                  Spacing, BorderRadius, Shadows, Opacity, Animations, IconSize, CardCatalog
├── Styles/        Buttons/ChipButtonStyle, SurfaceButtonStyle, ContentButtonStyle, ControlFeedback
│                  Progress/LoadingDotsStyle, ControlStyleMetrics
├── Components/    Artwork/, Feedback/, Labels/, Layout/
└── Extensions/    Color++ (internal helpers)
```

## Foundations (tokens)

All tokens hang off one namespace, `public enum DesignSystem {}`, grouped by role:

```swift
DesignSystem.Colors.Text.primary          // semantic role → a primitive
DesignSystem.Colors.Background.canvas
DesignSystem.Spacing.space5               // numeric scale
DesignSystem.BorderRadius.medium
DesignSystem.Animations.default
```

- Two color layers: `ColorPrimitives` (raw palette, never used by features) and `Colors` (semantic roles that features use).
- Typography tokens are relative to Dynamic Type text styles and applied with one modifier that sets font, tracking and line height:

```swift
Text(movie.title)
    .designTypography(DesignSystem.Typography.headline2)
    .foregroundStyle(DesignSystem.Colors.Text.primary)
```

- Effects that need several properties are value tokens plus a modifier: `.designShadow(DesignSystem.Shadows.artwork)`.
- Tokens are values, never views. A token namespace never returns `some View`.

## Styles: XStyle conformances

Restyle native controls by conforming to SwiftUI's style protocols (`ButtonStyle`, `ProgressViewStyle`, `ToggleStyle`, `LabelStyle`, `TextFieldStyle`, …), then expose each style as a static member so call sites use dot syntax. Features keep using `Button`, `ProgressView` and `NavigationLink`; only the style changes.

```swift
/// Compact capsule actions, e.g. "See All" or a filter.
public struct ChipButtonStyle: ButtonStyle {
    public enum Appearance: Sendable {
        case tonal
        case text
    }

    fileprivate let appearance: Appearance

    public func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .designTypography(DesignSystem.Typography.labelSmall)
            .padding(.horizontal, DesignSystem.Spacing.space4)
            .frame(minHeight: DesignSystem.ControlStyle.chipHeight)
            .background {
                if appearance == .tonal {
                    Capsule().fill(DesignSystem.Colors.Surface.base)
                }
            }
            .modifier(ControlFeedback(isPressed: configuration.isPressed))
    }
}

extension ButtonStyle where Self == ChipButtonStyle {
    public static var chip: Self { Self(appearance: .tonal) }

    public static func chip(_ appearance: ChipButtonStyle.Appearance) -> Self {
        Self(appearance: appearance)
    }
}
```

```swift
Button(.searchActionRetry, action: retry)
    .buttonStyle(.chip)

ProgressView()
    .progressViewStyle(.loadingDots)

NavigationLink(value: DiscoverDestination.movie(id: movie.id)) {
    MoviePosterCard(movie: movie)
}
.buttonStyle(.content)
```

Rules for styles:
- Name the type for its look plus `Style` (`ChipButtonStyle`, `SurfaceButtonStyle`, `LoadingDotsStyle`), and the static member after its role (`.chip`, `.filled`, `.filledProminent`, `.content`, `.loadingDots`).
- A variant is a parameter on one style (`.chip(.text)`), not a new type.
- Shared behavior across styles (press scale, disabled dimming) is an internal `ViewModifier` (`ControlFeedback`) that reads `configuration.isPressed` and `@Environment(\.isEnabled)`.
- Measurements shared by styles live in a token namespace (`DesignSystem.ControlStyle.chipHeight`).
- Keep native behavior: let `.disabled`, roles and the environment drive the style rather than adding parameters for them.

## Components

Generic views that features compose, taking plain values and view builders:

| Component | Takes |
|---|---|
| `Rail(_ title:content:accessory:)` | `LocalizedStringResource` title, cards, optional accessory |
| `StateContent(_ title:illustration:message:actions:)` | empty, error and offline states |
| `RemoteImage`, `MediaArtwork`, `PersonPortrait` | `URL?` |
| `PosterCaption`, `RatingBadge`, `TagLabel` | `String`, `Double`, optional `Text` subtitle |
| `FlowLayout` | a `Layout` for wrapping content |

- Titles are `LocalizedStringResource`, so a feature passes its own generated symbol and the string resolves from the feature's bundle: `Rail(.discoverRailTrendingTitle) { … }`.
- A component with an actions slot styles its own actions, so callers pass plain controls: `StateContent` applies `.filledProminent`, and `SurfaceButtonStyle` lays out a `Label`'s icon and title. An error view writes `Button(.discoverActionRetry, systemImage: "arrow.clockwise", action: retry)` with no style.
- Components never fetch, cache or decide policy. An image session or cache is injected by the app through the environment.
- A view that renders a domain type and is used by several features belongs in CoreUI (`MovieSummaryCard`), built from DesignSystem parts. A view used by one feature stays in that feature.

## Rules

- Domain-free: no feature models, services, destinations or `XCore` imports. Prefer generic names (`PersonPortrait`, `MediaArtwork`) over a feature's wording.
- No user-facing strings of its own. Previews use `Text(verbatim:)` and `LocalizedStringResource(stringLiteral:)`.
- Export icons at the size the design draws them, such as a 24-point icon frame as a 24-point asset. Then `Image(resource)` needs no `.resizable().frame(…)`, and the icon works as is in `Label(_:image:)` and `Button(_:image:action:)`.
- Public API is tokens, styles, components and modifiers only.
- Every style and component has a preview on `DesignSystem.Colors.Background.canvas`.
