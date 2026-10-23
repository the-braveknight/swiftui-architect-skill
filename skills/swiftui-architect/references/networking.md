# Networking

One package talks to the API. It generates the API's types and client from an OpenAPI document with Apple's [swift-openapi-generator](https://github.com/apple/swift-openapi-generator), wraps the generated client in the app's own client, and classifies every failure once. Only the Live targets, XTests and the app import it.

## Contents
- The package
- Generating the API
- The client
- Errors and cancellation
- Testing

## The package

```
Packages/Networking/
├── Package.swift
├── Reference/                         the full upstream spec, for looking things up
├── Sources/Networking/
│   ├── openapi.yaml                   the endpoints and schemas the app uses
│   ├── openapi-generator-config.yaml
│   ├── TMDBClient.swift               the app's client over the generated one
│   ├── TMDBError.swift                transport failures, classified once
│   └── Middleware/                    only what the API requires, such as auth
└── Tests/NetworkingTests/
```

```swift
dependencies: [
    .package(url: "https://github.com/apple/swift-openapi-generator", from: "1.10.0"),
    .package(url: "https://github.com/apple/swift-openapi-runtime", from: "1.8.0"),
    .package(url: "https://github.com/apple/swift-openapi-urlsession", from: "1.0.0"),
],
targets: [
    .target(
        name: "Networking",
        dependencies: [
            .product(name: "OpenAPIRuntime", package: "swift-openapi-runtime"),
            .product(name: "OpenAPIURLSession", package: "swift-openapi-urlsession"),
        ],
        swiftSettings: swiftSettings,
        plugins: [
            .plugin(name: "OpenAPIGenerator", package: "swift-openapi-generator")
        ]
    ),
]
```

## Generating the API

The build plugin, upstream's recommended setup, generates the code at build time from the two files in the target's source folder, so generated code is never committed. Xcode asks to trust the plugin once; CI skips plugin validation (`-skipPackagePluginValidation`).

```yaml
# openapi-generator-config.yaml
generate:
  - types
  - client
accessModifier: public
namingStrategy: idiomatic
```

- `openapi.yaml` describes only what the app uses. Trim a large upstream spec by hand (keeping the original in `Reference/`), or narrow it with the config's `filter:`. Add an endpoint to the spec before using it.
- `idiomatic` names generate Swift-style types and members (`releaseDate`, not `release_date`).
- The generated types are `public` because the Live targets in other packages map them. That makes them reachable, not shared: they never appear in an XCore model, a service protocol or anything above the Live targets.

## The client

`TMDBClient` is a `Sendable` struct over the generated `Client`. It's the app's map of the API: one method per operation the app calls, returning the generated schema types for XLive to map.

```swift
public struct TMDBClient: Sendable {
    private let client: Client

    public func movieDetails(id: Int) async throws -> Components.Schemas.MovieDetails {
        switch try await client.movieDetails(path: .init(movieId: id)) {
        case .ok(let response):
            try response.body.json
        case .undocumented(let status, _):
            throw TMDBError.unexpectedStatus(status)
        }
    }
}
```

- Methods keep the spec's operation names (`movie-details` → `movieDetails(id:)`), so the client reads like the spec it wraps. Swift naming starts at the domain ([domain](domain.md#naming-services-and-their-methods)).
- Each method switches over every documented output plus `.undocumented`, with no `default`, so a response newly added to the spec is a compile error where it must be handled.
- Unwrap envelopes here (`{ "data": … }`), so XLive maps the payload itself.
- The app builds the client with a URL session; tests build it over a stub transport.
- Middleware is for what every request needs and the API requires, such as an auth header or a language parameter. Add none the app doesn't need (logging, retries).

## Errors and cancellation

Every client method rethrows what fails as either `TMDBError` or `CancellationError`. XLive maps `TMDBError` into the feature's error, and view models treat `CancellationError` as "nothing happened".

```swift
public enum TMDBError: Error, Equatable {
    /// No connection, or it dropped during the request.
    case offline
    /// The request didn't complete for another reason, such as a timeout.
    case requestFailed
    /// The server answered with something that couldn't be decoded.
    case invalidResponse
    /// A status the spec doesn't document, such as 404.
    case unexpectedStatus(Int)
}
```

- Failures are classified once, in the client, so live services see only `TMDBError` and `CancellationError`, never `URLError` or the runtime's errors.
- Cancellation must arrive as `CancellationError`: URLSession reports a cancelled task as `URLError(.cancelled)`, and a request can fail some other way once its task is cancelled. Left as failures, a screen left mid-load would come back showing an error.
- Only a lost connection is `.offline`. A timeout doesn't prove the device is offline.
- The generated runtime wraps failures in `ClientError`; classify what it wraps.

## Testing

`NetworkingTests` drives the client through a stub `ClientTransport`: the requests it builds, the responses it decodes, and the classification, cancellation included. Features never test the network; their XLive tests map hand-built generated types ([testing](modules.md#testing)).
