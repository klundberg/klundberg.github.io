---
title: Encapsulating Value Semantics, Part 2
tags:  swift value-semantics
category: programming
---

In my [previous post]({{ site.baseurl }}{% post_url 2016-12-20-encapsulating-value-semantics %}) in this series, I gave a brief overview of how value semantics work along with a couple of good resources for more in-depth details on how you get value semantics with your types in Swift. In this post I want to fulfill the promise I made on showing a way to nicely encapsulate value semantics in your types, so that you can reduce the boilerplate needed to implement this functionality.

First off, in the interest of time I omitted a detail from my last post that the objc.io guys raised in their video, which is that `isKnownUniquelyReferenced` can only work with pure Swift classes. I presume that this is due to the fact that in Objective-C, non-ARC classes can muck around with `retainCount` or even implement their own reference counting schemes that Swift cannot know about. The way to solve this is to wrap references in a swift-only wrapper class, which then lets `isKnownUniquelyReferenced` work as expected. If `Foo` subclassed `NSObject` in the previous post's example, the full implementation would have to look like this:

```swift
class Box<T> {
  let unbox: T
  init(_ value: T) {
    unbox = value
  }
}

class Foo: NSObject {
  var count: Int = 0
}

struct Counter {
  var _counter = Box(Foo())

  mutating func increment() {
    // copy the reference only if necessary
    if !isKnownUniquelyReferenced(&_counter) {
      var old = _counter
      _counter = Box(Foo())
      _counter.unbox.count = old.unbox.count
    }

    _counter.unbox.count += 1
  }

  var value: Int {
    return _counter.unbox.count
  }
}
```

That `Box` class adds even more boilerplate since we have to constantly reference the value inside of it in order to operate on the data at all.

# Identifying the pieces

The first step in abstracting out what we want is to identify the different ways we need to operate on our reference types. There are two general operations we need to perform: mutating data and creating the copy when necessary. This can be done with two closures, of the following types:

```swift
/// given a reference to a T, update its contents
typealias UpdateClosure<T> = (T) -> Void
/// given a reference to a T, return a new copy of that reference
typealias CopyClosure<T> (T) -> T
```

So let's start constructing a type that can encapsulate this, which we'll call `CopyOnWrite`. Firstly, we need to initialize it:

```swift
struct CopyOnWrite<T: AnyObject> {
  var reference: Box<T>
  private let copy: CopyClosure<T>

  init(reference: T, copy: @escaping CopyClosure<T>) {
    self.reference = Box(reference)
    self.copy = copy
  }
}
```

Simple enough so far. Copying is an act that only needs to be done once per type, so we can provide that here in the init method so that we don't duplicate the copying logic every time we want to modify a property in our reference type.

The generic type `T` is constrained to be `AnyObject`, since it makes no sense to use this with value types like structs and enums, and we won't be able to call `isKnownUniquelyReferenced` on our type otherwise.

Before we continue, the fact that we use `Box` to wrap our reference is really an implementation detail that we shouldn't burden consumers of this API with. If we left it as is, code using this type would have to say `thing.reference.unbox.blah` whenever they wanted to read from a property in the reference. To alleviate this a bit, we should make the actual boxed reference private, and provide a convenience property instead:

```swift
private var _reference: Box<T>

var reference: T {
  return _reference.unbox
}
```

So far, so good.

Now we need to incorporate the updating half of this behavior, which will allow the caller to provide a way to update the reference's value however they see fit, and the struct can decide whether or not it actually needs to copy the reference first:

```swift
mutating func update(_ update: UpdateClosure<T>) {
  if !isKnownUniquelyReferenced(&_reference) {
    _reference = Box(self.copy(_reference.unbox))
  }

  update(_reference.unbox)
}
```

This will unbox the reference from the swift-only wrapper, pass that along, and box up the result of the copy again transparently so that the caller does not need to worry about doing so themselves ad nauseum. We then call the given update closure to actually update the reference's contents for us.

The function must be `mutating` since we're updating a mutable value in the struct, which will make it impossible to call this function unless the caller is also in a `mutating` function or property (one of the nice ancillary things about value types).

It should be noted here that there's nothing preventing the copy closure from violating the contract of this type by just returning `_reference.unbox` immediately. Enforcing this is impossible in Swift's type system, and the only way we could even attempt to enforce this rule would be by trapping at runtime if the new reference has an identical pointer as the old one:

```swift
let old = _reference.unbox
_reference = Box(self.copy(_reference.unbox))
precondition(old !== reference)
```

This is a bit ugly, and runtime failures like this may not consistently be hit if the copy closure never needs to run, so we can omit it for now.

All together, the type looks like this:

```swift
struct CopyOnWrite<T: AnyObject> {
  private var _reference: Box<T>
  private let copy: CopyClosure<T>

  init(reference: T, copy: @escaping CopyClosure<T>) {
    self._reference = Box(reference)
    self.copy = copy
  }

  var reference: T {
    return _reference.unbox
  }

  mutating func update(_ update: UpdateClosure<T>) {
    // copy the reference only if necessary
    if !isKnownUniquelyReferenced(&_reference) {
      _reference = Box(self.copy(_reference.unbox))
    }

    update(_reference.unbox)
  }
}
```

Simple, to the point, and short. Very nice!

Applying this to the original `Counter` example yields the following code:

```swift
class Foo: NSObject {
  var count: Int = 0
}

struct Counter {
  var _counter = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }

  mutating func increment() {
    _counter.update { $0.count += 1 }
  }

  var value: Int {
    return _counter.reference.count
  }
}
```

Much better! the only code that is in here is the actual logic needed to accomplish the task, while the code to handle the copy on write optimization is hidden away within the new struct we made. The `count` field within counter could still be modified outside of an `update` call, but there's not a good way to guard against that completely: it will be up to the consumer of this API to know when it is appropriate to access `reference` directly, and when `update` should be called.

# Further Improvements?

What we have so far is pretty nice, but it's still a bit flawed. What if we wanted to add multiple properties of the same type to a struct? We would be forced to provide the same copy closure each time we create an instance of `CopyOnWrite`:

```swift
struct Counter {
  var first = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }
  var second = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }
  // ...
}
```

Yuck! We have to provide the exact same copy closure each time we create an instance of `Foo`, which isn't good. So, let's extract this copying behavior into its own `init` method on `Foo`:

```swift
extension Foo {
  init(copying original: Foo) {
    self.init()
    self.count = original.count
  }
}

struct Counter {
  var first = CopyOnWrite(reference: Foo(), copy: Foo.init(copying:))
  var second = CopyOnWrite(reference: Foo(), copy: Foo.init(copying:))
  // ...
}
```

That's better, but we have to specify the copying initializer each time, which still feels like needless repetition. To solve this, we can create a `Copyable` protocol to represent all copyable types:

```swift
protocol Copyable: class {
  init(copying original: Self)
}

class Foo: NSObject, Copyable  {
  var count: Int = 0

  required convenience init(copying original: Foo) {
    self.init()
        self.count = original.count
    }
}
```

We constrain the `Copyable` protocol to only be applicable to classes, since structs and enums already are copied by default when you assign them to new variables, so it wouldn't make much sense for those types to conform to it. We also must conform to `Copyable` within the main class declaration here, since Swift 3 unfortunately doesn't allow you to specify required initializers within a class extension. This could be worked around by using a `copy()` method instead of an initializer in the `Copyable` protocol, but I personally like the feel of the `init` method better so I'll roll with it here.

Now, we can extend `CopyOnWrite` for all cases where `T` conforms to `Copyable` to provide a new, simpler initializer there. This will make assigning value semantics to a reference in a struct much easier if that reference conforms to `Copyable`:

```swift
extension CopyOnWrite where T: Copyable {
    init(reference: T) {
        self.init(reference: reference, copy: T.init(copying:))
    }
}
```

And now with this, we can finally clean up the original example:

```swift
class Foo: NSObject, Copyable  {
  var count: Int = 0

  required convenience init(copying original: Foo) {
    self.init()
    self.count = original.count
  }
}

struct Counter {
  var _counter = CopyOnWrite(reference: Foo())

  mutating func increment() {
    _counter.update { $0.count += 1 }
  }

  var value: Int {
    return _counter.reference.count
  }
}
```

If we have to add many properties that hold values of the same type, we just have to conform the type to `Copyable`, wrap all construction or assignment of those references into the `CopyOnWrite` struct, and that's basically it!

# Conclusion

We've done a lot to extract all the repetitive boilerplate code into a reusable component that can be applied anywhere you want to have value semantics in your structs, while achieving the performance benefits that come with copy-on-write behavior. I hope that you might find this useful, and if you have any feedback please feel free to contact me on [Twitter](https://twitter.com/kevlario) or via [email](mailto:{{ site.author.email }}). I have a gist of a tweaked version of the final code for `CopyOnWrite` available [here](https://gist.github.com/klundberg/9faf7b934fc2ecbaa8d9fa414905ba7c) as well if you want to use this in your own project.

Until next time!
