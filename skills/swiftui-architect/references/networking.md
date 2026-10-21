# Networking

One package talks to the API. It generates the API's types and client from an OpenAPI document with Apple's [swift-openapi-generator](https://github.com/apple/swift-openapi-generator), wraps the generated client in the app's own client, and classifies every failure once. Only XLive targets, XTests and the app import it.

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

Like every package, it has no default actor isolation: generated code doesn't build under MainActor default isolation.

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
- The generated types are `public` because XLive targets in other packages map them. That makes them reachable, not shared: they never appear in an XCore model, a service protocol or anything above XLive.

## The client

`TMDBClient` is a `Sendable` struct over the generated `Client`. It's the app's map of the API: one method per operation the app calls, returning the generated schema types for XLive to map.

```swift
public struct TMDBClient: Sendable {
    private let client: Client

    public init(session: URLSession = .shared) throws {
        try self.init(transport: URLSessionTransport(configuration: .init(session: session)))
    }

    /// Tests pass a stub transport.
    init(transport: any ClientTransport) throws {
        client = Client(
            serverURL: try Servers.Server1.url(),
            transport: transport,
            middlewares: []   // only what the API requires, such as auth
        )
    }

    public func movieDetails(id: Int) async throws -> Components.Schemas.MovieDetails {
        try await classifyingErrors {
            switch try await client.movieDetails(path: .init(movieId: id)) {
            case .ok(let response):
                try response.body.json
            case .undocumented(let status, _):
                throw TMDBError.unexpectedStatus(status)
            }
        }
    }

    /// Rethrows what a request throws as `TMDBError` or `CancellationError`.
    private func classifyingErrors<Value>(_ request: () async throws -> Value) async throws -> Value {
        do {
            return try await request()
        } catch {
            throw TMDBError.classify(error)
        }
    }
}
```

- Methods keep the spec's operation names (`movie-details` → `movieDetails(id:)`), the standard OpenAPI naming, so the client reads like the spec it wraps. Swift naming practice starts at the domain: services and their methods ([domain](domain.md#naming-services-and-their-methods)).
- Each method switches over every documented output plus `.undocumented`, with no `default`, so a response newly added to the spec is a compile error where it must be handled.
- Unwrap envelopes here (`{ "data": … }`), so XLive maps the payload itself.
- Middleware is for what every request needs and the API requires, such as an auth header or a language parameter. Add none the app doesn't need (logging, retries).

## Errors and cancellation

The client rethrows every failure as either `TMDBError` or `CancellationError`. XLive maps `TMDBError` into the feature's error, and view models treat `CancellationError` as "nothing happened".

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

extension TMDBError {
    /// A cancelled request surfaces as `URLError.cancelled`, or as whatever failed once its task was cancelled,
    /// so both become `CancellationError`.
    static func classify(_ error: any Error) -> any Error {
        if Task.isCancelled {
            return CancellationError()
        }
        var underlying = error
        while let clientError = underlying as? ClientError {
            underlying = clientError.underlyingError
        }
        switch underlying {
        case let error as TMDBError:
            return error
        case is CancellationError:
            return CancellationError()
        case let error as URLError:
            switch error.code {
            case .cancelled: return CancellationError()
            case .notConnectedToInternet, .networkConnectionLost: return TMDBError.offline
            default: return TMDBError.requestFailed
            }
        default:
            return TMDBError.invalidResponse
        }
    }
}
```

- The runtime wraps transport, decoding and middleware failures in `ClientError`; unwrap it before classifying.
- Cancellation must arrive as `CancellationError`. URLSession reports a cancelled task as `URLError(.cancelled)`; left as a failure, a footer scrolled away mid-load would come back showing a retry, and a screen left mid-load would come back showing an error.
- Only a lost connection is `.offline`. A timeout doesn't prove the device is offline.

## Testing

`NetworkingTests` drives the client through a stub `ClientTransport`: the requests it builds, the responses it decodes, and the classification, cancellation included. Features never test the network; their XLive tests map hand-built generated types ([testing](modules.md#testing)).
