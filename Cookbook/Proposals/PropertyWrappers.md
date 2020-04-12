# Using Property wrappers to reduce decoding boilerplate

* Author(s): Ilya Puchka
* Review Manager: -

## Introduction

Instead of using custom decoding methods as we do now we propose to use property wrappers which allows to write custom decoding code only for these wrappers and leave the rest for compiler.
As a matter of fact we already have such "wrappers" in our code base, but they are not annotated with `@propertyWrapper` as they were introduced to our code base before this feature became available in Swift. Examples of such wrappers are `ISO8601Date`, `NestedKeyDecodable`, `FailableDecodable`, `NotEmptyDecodable` etc.

Ideally with this approach we would be able to get rid of all custom decoding/encoding methods. This not only decreases amount of boilerplate but also makes the code easier to understand.

## Motivation

Quite often in our code we have to implement custom decoding/encoding methods when one or few fields do not follow convensions for which compiler can generate code for us.
Some of the examples of such cases:

 - default value

 For that we are using `decodeIfPresent(...) ?? defaultValue`
 
 - safe decoding

 `decodeIfPresent` still fails when value is of a wrong type (i.e. unknown enum case) so we use custom `decodeSafelyIfPresent` helper method that returns `nil` in case of error
 
 - dates decoding

 usually dates are decoded as strings and then flat-mapped using some date formatting helper method
 
 - decoding nested values into self

 sometimes we need to extract nested keys without preserving intermediate structure, for which we use custom `NestedKeyDecodable` helper and custom parsing methods
 
 - decoding self values into nested objected

 opposite to the previous keys sometimes we decode nested objects from the flat json structure, for which we directly pass decoder to `init(decoder:)`
 
All these cases require us to implement decoding for all other properties as well, which could otherwise be decoded by compiler generated code.
 
### Detailed design

The Swift compiler is capable of generating decoding code for properties wrapped with property wrappers in a non disruptive way (i.e. it does not require changing coding keys to match names of wrapper properties generated by the compilers). For that property wrappers should implement coding protocols. Additionally it's possible to extend standard library decoding containers with decoding methods for specific types of wrappers without writing decoding code for the whole type - the container will pick up extension methods if the types in the signature match. This way we can customise decoding for non-standard cases in various ways.

The simplest example of a property wrapper would be if we make an `ISO8601Date` property wrapper. This only requires to add a `@propertyWrapper` annotation to this type. It already implements decoding for dates so we don't need to change that. What changes is only how we declare properties of this type.

Before:

```swift
struct Appointment: Decodable {
  let time: Date
	
  init(from decoder: Decoder) throws {
    let values = try decoder.container(keyedBy: CodingKeys.self)
    time = try values.decode(ISO8601Date.self, forKey: .time)
  }
}
```

After:

```swift
struct Appointment: Decodable {
  @ISO8601Date
  let time: Date
}
```

Reference: https://speakerdeck.com/alisoftware/and-thats-a-wrap?slide=25

Another example is pruned/strict array decoding. The pruned strategy means that any array element that fails decoding will be discarded whereas with the strict strategy the whole decoding will fail on any failure in decoding of any array element (default behaviour). To implement prune strategy we use `FailableDecodable` that decodes its value with `value = try? container.decode(T.self)` or `decodeArraySafely` method. With a property wrapper we wouldn't need to to call this method manually or change the type of the property. We can also implement more strategies, like non-empty, using the same property wrapper.

Before:

```swift
struct DemographicsListDTO: Decodable {
  let demographics: [DemographicDTO]
	
  init(from decoder: Decoder) throws {
    let values = try decoder.container(keyedBy: CodingKeys.self)
    let demographics = try values.decodeArraySafely([DemographicDTO].self, forKey: .demographics)
  }
}
```

After:

```swift
struct DemographicsListDTO: Decodable {
  @ArrayDecodable<Prune, DemographicDTO>
  let demographics: [DemographicDTO]
}
```

Providing defaults for absent values has slightly tricky ergonomics, as this approach does not allow to pass arbitrary parameters to property wrapper initialisers when they are being decoded (so `@Default(value: true)` is possible programmatically but won't affect decoding). So instead of using values we use types which would correspond to common defaults, i.e. `True`, `False`, `Empty` etc:

```swift
struct Product: Codable {
  var name: String
  
  @DefaultDecodable<Empty>
  var description: String
  
  @DefaultDecodable<True>
  var isAvailable: Bool
}
```

Reference: https://github.com/gonzalezreal/DefaultCodable

Nested keys are a bit more tricky, but still possible:

```swift
struct InboxMessage: Decodable {
  @NestedDecodable<Bool, IsReadKeys>
  var isRead: Bool // { "status": { "is_read": true } }
  
  enum CodingKeys: String, CodingKey {
    case isRead = "status"
  }
  
  enum IsReadKeys: String, CodingKey, CaseIterable {
    case isRead = "is_read"
  }
}
```

Here we specify an additional coding keys type that will be used to decode `isRead`. Then the container extension method takes care of traversing json using these additional keys and their order (via CaseIterable) as a key path.

When needed we can provide Encoding and Codable wrappers, though in general we have either encodable or decodable types. We do have encoding implemented manually for types used by mock server in UI tests, so providing Codable wrappers for properties of these types would also decrease code in tests and will make it easier to add more types to mock server.


## Impact on existing codebase

To replace each helper methods property wrappers would be introduced via separate change to simplify transition and testing.


## Alternatives considered

1. We can come up we a single property wrapper to describe decoding strategy and provide the actual strategy as a generic parameter:

```swift
@Decoding<Default<Empty>>
@Decoding<Array<Prune>>
@Decoding<Nested<IsReadKeys>>
```

This might be a good way to unify the approach, but it's a bit harder to read and we would need to figure out how to constrain particular strategies to particular types of wrapped values, which seems to be easier with a more straight forward approach.

2. [KeydCodable](https://github.com/dgrzeszczak/KeyedCodable) provides an alternative approach where custom keys type and custom decoders are used along with some property wrapper. While it's a nice solution it requires much more code to implement (most of it - boilerplate) or we would need to depend on this library which is undesirable when the same can be achieved with simple property wrappers.

### References

https://github.com/gonzalezreal/DefaultCodable

https://github.com/GottaGetSwifty/CodableWrappers

https://github.com/dgrzeszczak/KeyedCodable