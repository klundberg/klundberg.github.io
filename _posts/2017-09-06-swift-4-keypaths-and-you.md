---
title: Swift 4 KeyPaths and You
category: programming
tags: swift
date: 2017-09-06 21:20 -0500
---

Swift 4 is almost upon us, and I thought I would explore one of its features that I haven't had the opportunity to that much: KeyPaths. There's a lot of interesting nuance in here that I previously didn't realize existed which I'd love to share.

In short, KeyPaths are a type-safe way to separate referencing a type's property from evaluating that property and getting a result back. You can already do that today in Swift 3 with functions, but until Swift 4 is released you can't do so with properties without wrapping them in a closure, or with using the old unsafe `#keyPath()` syntax if your code is running on ios/mac/etc:

```swift
struct Person {
  let name: String

  func greet() {
    print("Hello \(name)!")
  }
}

let p = Person(name: "Samus")
let greeter = p.greet // stores the method without evaluating it.
greeter() // calls the stored method

// this is the only way in Swift 3.1 and below to defer evaluating the name property of a Person.
let getName = { (p: Person) in p.name }
print(getName(p)) // evaluate the property
```

With Swift 4 you can rewrite the last two lines above like so:

```swift
let getName = \Person.name
print(p[keyPath: getName])

// or just this:
print(p[keyPath: \Person.name])
```

`\Person.name` is the way to construct a `KeyPath<Person, String>` where the first generic parameter is the root type (what type we are querying), and the second generic parameter is the type of the value we are asking for. Using this, we can ask for the value of the `name` property from any instance of `Person`, but without the overhead of defining a closure every time we might want to ask for it. You can query more than one level as well. If you wanted to get a `KeyPath` for the length of a person's name, you could write `\Person.name.count`.

In this example we created a `KeyPath` which is read-only, because the `name` property on `Person` was defined as a `let`. What happens if we change it to a `var`?

```swift
struct Person {
  var name: String

  func greet() {
    print("Hello \(name)!")
  }
}

let kp = \Person.name
let p = Person(name: "Samus")
p[keyPath: kp] = "Ridley"
p.greet() // prints "Hello Ridley!"
```

In this case, since `name` is defined as a `var`, `\Person.name` is a `WritableKeyPath<Person, String>` which means we can use the subscript to set values in addition to getting them. If we tried to use the subscript setter for a `let` property we'd actually get a compiler error since settable subscripts are not defined for the plain `KeyPath` type, only `WritableKeyPath` which is a pretty nice way to make sure you can't do something that isn't possible.

There are more KeyPath types than just `KeyPath` and `WritableKeyPath` as well. If you look at the [Swift Evolution proposal for KeyPaths](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md) or the generated interface for the `KeyPath` classes in Xcode, you'll see that the `KeyPath` types are actually a 5-deep linear class hierarchy:

```
AnyKeyPath
    |
    v
PartialKeyPath<Root>
    |
    v
KeyPath<Root, Value>
    |
    v
WritableKeyPath<Root, Value>
    |
    v
ReferenceWritableKeyPath<Root, Value>
```

We went over the third and fourth type which are pretty easy to grok, but the others may not be. `ReferenceWritableKeyPath` is a subclass of `WritableKeyPath` which means that it can also be used both in a KeyPath getter and setter, but that's the type of KeyPath you get if you referenced a mutable property on a class:

```swift
class BankAccount {
  var balance: Decimal = 0
  var owner: Person
}

// Creates a ReferenceWritableKeyPath<BankAccount, Decimal>:
let kp = \BankAccount.balance
```

`ReferenceWritableKeyPath` is necessary so that the compiler can know whether it's safe to allow you to use a KeyPath setter on a value that's stored in a constant. If that value is a `class` type, then mutating the value's properties is possible in any context, therefore this KeyPath is the only one that can be used for it. If it was a `struct` or other value type stored in a constant, you would not be able to set something with the `keyPath:` subscript since that would change the value itself, which is not allowed for `let` constants.

Moving up the tree we have `PartialKeyPath<Root>`. This KeyPath has a concrete root, but an unknown value type at compile time. It could be useful in cases where you might want to store all of a type's KeyPaths in a generic data structure or if you want to do other things that don't rely on a KeyPath's evaluated value, since the resulting value is returned as the `Any` type:

```swift
struct Person {
  var name: String
  let birthdate: Date
}

let kp: PartialKeyPath<Person>: \Person.name
type(of: person[keyPath: kp]) // returns Any

let personPaths: [PartialKeyPath<Person>] = [
    \Person.name,
    \Person.birthdate,
] // only possible with PartialKeyPath since name and birthdate are different types
```

Finally there's the base class for everything, `AnyKeyPath`. Here we don't know the type of the value OR the type of the root at compile time. These can be queried at runtime though if necessary:

```swift
let kp: AnyKeyPath = \Person.birthdate

kp.rootType // evaluates to Person.self
kp.valueType // evaluates to Date.self
```

These properties are available to all the subclasses as well, though they're not very interesting once you get to `KeyPath` since you already have the compile time generic type parameters at that point.

## Combining KeyPaths

There's another cool thing you can do with KeyPaths: combining them together! Every KeyPath type has a number of `appending` methods that let you stick together two KeyPaths to make one mega-KeyPath:

```swift
let accountOwnerPath = \BankAccount.owner
let namePath = \Person.name

let accountOwnerNamePath = accountOwnerPath.appending(namePath) // returns KeyPath<BankAccount, String>

let account: BankAccount = ...
account[keyPath: accountOwnerNamePath] // returns the name of the account owner
```

You can only append two KeyPaths if the `Value` of the first one in the chain is of the same type as the `Root` of the second one in the chain. If you try to append two wholly-unrelated KeyPaths, you'll get different behavior depending on which type of KeyPath you're working with. If you're working with `KeyPath` and its subclasses exclusively, the compiler can check your work and give you a compiler error if the types don't match up nicely. However if you're working with `AnyKeyPath` or `PartialKeyPath`, appending those together with any KeyPath type will result in returning an optional KeyPath, where you'll get nil at runtime if the KeyPaths' types don't line up properly.

Combining KeyPaths has some other interesting behavior around how the types of the KeyPaths you combine affect the KeyPath you get back. For instance, appending a `KeyPath` and a `WritableKeyPath` gives you a read-only `KeyPath` since it's not possible to mutate a property in the normal case either:

```swift
struct Person {
  let birthPlanet: Planet
}

struct Planet {
  var name: String
}

var person: Person = ...
person.birthPlanet.name = "SR388" // error: can't mutate birthPlanet.name
person[keyPath: \Person.birthPlanet.name] = "Zebes" // error for the same reason
```

When working with `ReferenceWritableKeyPath` though, it doesn't always take the least strict version. If you append a `ReferenceWritableKeyPath` to the end of anything else (except `AnyKeyPath`), the result is a `ReferenceWritableKeyPath` since the KeyPath can safely mutate anything at the end of the chain if there's reference semantics somewhere along the way. If you append a read-only `KeyPath` to the end of a `ReferenceWritableKeyPath` though, it's still just a `KeyPath` since the tail end of the chain is still immutable.

The full table of ways you can combine KeyPaths and get different types back is below, for reference:

| First | Second | Result |
| ---   | ---    | ---    |
| `AnyKeyPath` | Anything | `AnyKeyPath?` |
| `PartialKeyPath` | `AnyKeyPath` or `PartialKeyPath` | `PartialKeyPath?` |
| `PartialKeyPath` | `KeyPath` or `WritableKeyPath` | `KeyPath?` |
| `PartialKeyPath` | `ReferenceWritableKeyPath` | `ReferenceWritableKeyPath?` |
| `KeyPath` | `AnyKeyPath` or `PartialKeyPath` | 💥 Not possible 💥 |
| `KeyPath` | `KeyPath` or `WritableKeyPath` | `KeyPath` |
| `KeyPath` | `ReferenceWritableKeyPath` | `ReferenceWritableKeyPath` |
| `WritableKeyPath` | `AnyKeyPath` or `PartialKeyPath` | 💥 Not possible 💥 |
| `WritableKeyPath` | `KeyPath` | `KeyPath` |
| `WritableKeyPath` | `WritableKeyPath` | `WritableKeyPath` |
| `WritableKeyPath` | `ReferenceWritableKeyPath` | `ReferenceWritableKeyPath` |
| `ReferenceWritableKeyPath` | `AnyKeyPath` or `PartialKeyPath` | 💥 Not possible 💥 |
| `ReferenceWritableKeyPath` | `KeyPath` | `KeyPath` |
| `ReferenceWritableKeyPath` | `WritableKeyPath` or `ReferenceWritableKeyPath` | `ReferenceWritableKeyPath` |

You'll notice that you can't append `KeyPath` or its subclasses with `AnyKeyPath` or `PartialKeyPath`, unless you upcast the first KeyPath to one of those type erased variants too. I'm not sure why these weren't included. My best guess is that the swift team didn't want developers to accidentally move into a type-erased world when they didn't intend to, and upcasting would force developers to consciously opt-in to that behavior.

## And so much more!

Well, three things more. You can use optional chaining to model optional properties in the same way it works by directly referencing them:

```swift
struct Person {
  var address: Address?
}
struct Address {
  var fullAddress: String
}

let kp = \Person.address?.fullAddress // returns WritableKeyPath<Person, String?>
```

However I haven't found a way to append KeyPaths with optionals somewhere in the chain. If a KeyPath ends in an optional Value, it's unclear to me how you would append a KeyPath to that since you can't (as far as I know) create a KeyPath that would lift its types up to an optional `Root` and `Value` directly. I expect explicit compiler or standard library support would need to be added to make this work.

You can also ostensibly use subscripting in KeyPaths:
```swift
struct Person {
  var previousAddresses: [Address]
}

let kp = \Person.previousAddresses[0].fullAddress // WritableKeyPath<Person, String>
```

However as of the swift snapshot in Xcode 9 beta 6, this isn't working (with an actually useful error message saying that it's not implemented yet).

There's also ostensibly a way to use inferred types in key paths:

```swift
let p: Person = ...
p[keyPath: \.name] // \.name should evaluate to WritableKeyPath<Person, String>,
// referencing the name property on Person since we're calling it on an instance of Person
```

However this also appears to not be implemented yet (with a much less useful error message). I expect it will make it into Swift 4.1 if it's too late for it to get into Swift 4.0 (as of this article's publishing).

## Conclusion

KeyPaths are a really cool feature that I haven't heard discussed that much in the Swift developer community. I expect that more people will get excited about them once Swift 4 is released and people have more chances to play with them. There's a lot of cool stuff in Foundation that utilizes the feature too that's beyond the scope of this post (which I'll probably get to in the future).

I hope this was informative, and that it inspires you to find useful or interesting ways to take advantage of this feature.
