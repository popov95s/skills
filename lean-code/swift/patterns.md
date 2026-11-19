# Patterns: Swift

Which pattern fits which problem, in Swift. The rules are in [SKILL.md](../SKILL.md) and
[guide.md](guide.md).

| Problem | Pattern | Simpler form that may be enough |
| --- | --- | --- |
| Behaviour differs by kind, and cases call services | [Strategy](#strategy) | One kind needs none; data per kind is a `switch` returning values |
| Many stores differ only in value and file | [Generic store](#generic-store) | One store, when there is one value |
| Two drawings or modes differ only in colours or numbers | [Named variants](#named-variants) | A `Bool` that states a fact, such as `isOn` |
| The same view styling is repeated | [View modifier](#view-modifier) | One view, when it is used once |
| Callers repeat a system query and its filtering | [Reader](#reader) | The one call, when only one place makes it |

## Strategy

```swift
protocol Quoter { func quote() async throws -> Money }

struct CourierQuoter: Quoter {
    let rates: (String) async throws -> Rate
    let shipment: CourierShipment

    func quote() async throws -> Money {
        let rate = try await rates(shipment.postcode)
        return rate.base + rate.perKg * shipment.weightKg
    }
}

func quoter(for shipment: Shipment) -> any Quoter {
    switch shipment {
    case .postal(let postal): PostalQuoter(table: PostalRates.load, shipment: postal)
    case .courier(let courier): CourierQuoter(rates: CourierApi.rates, shipment: courier)
    case .locker(let locker): LockerQuoter(find: LockerApi.find, shipment: locker)
    }
}
```

One file per quoter, one picker, and the use case takes the `Quoter`. Tests build a quoter with
a fake closure.

## Generic store

Several stores that load a value, save it and fall back to empty are one generic type with a
file and a memory version. Per-value behaviour goes in a constrained extension.

```swift
protocol DocumentStore<Value>: Sendable {
    associatedtype Value: Codable & StartsEmpty
    func load() -> Value
    func save(_ value: Value)
}

struct FileDocumentStore<Value: Codable & StartsEmpty>: DocumentStore {
    private let file: JSONFile<Value>
    init(_ url: URL?) { file = JSONFile(url) }
    func load() -> Value { file.load() ?? Value() }
    func save(_ value: Value) { file.save(value) }
}

typealias PitchStore = any DocumentStore<[Pitch]>

extension DocumentStore where Value == [Pitch] {
    func upsert(_ pitch: Pitch) { save(load().filter { $0.id != pitch.id } + [pitch]) }
}
```

The memory version is a small class guarded by a lock, used by tests and previews.

## Named variants

```swift
struct FigureInk {
    let lit: Color
    let unlit: Color
    func color(isTrained: Bool) -> Color { isTrained ? lit : unlit }

    static func solid(tint: Color, secondary: Color) -> FigureInk { .init(lit: tint, unlit: secondary.opacity(0.55)) }
    static func ghost(tint: Color, secondary: Color) -> FigureInk { .init(lit: tint.opacity(0.22), unlit: tint.opacity(0.1)) }
}
```

The drawing code takes an ink and never asks whether it is the ghost.

## View modifier

```swift
enum ChipLayout { case pill, block }

private struct ChipStyle: ViewModifier {
    @Environment(\.theme) private var theme
    let isOn: Bool
    let layout: ChipLayout
    func body(content: Content) -> some View { /* font, padding, background, overlay */ }
}

extension View {
    func chipStyle(isOn: Bool, layout: ChipLayout = .pill) -> some View {
        modifier(ChipStyle(isOn: isOn, layout: layout))
    }
}
```

## Reader

When four places query the same system data and filter the same way, one type owns the query:

```swift
struct WatchSamples {
    let store: HKHealthStore
    func heart(in window: SampleWindow) async throws -> [HeartSample] { /* query, filter to the watch, map */ }
}
```

The window is a value (`start`, optional `end`), not two optional dates passed around.
