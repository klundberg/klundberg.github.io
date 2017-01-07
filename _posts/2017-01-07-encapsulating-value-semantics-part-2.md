---
title: Encapsulating Value Semantics, Part 2
tags:  swift value-semantics
category: programming
---

In my [previous post]({% post_url 2016-12-20-encapsulating-value-semantics %}) in this series, I gave a brief overview of how value semantics work along with a couple of good resources for more in-depth details on how you get value semantics with your types in Swift. In this post I want to fulfill the promise I made on showing a way to nicely encapsulate value semantics in your types, so that you can reduce the boilerplate needed to implement this functionality.

First off, in the interest of time I omitted a detail from my last post that the objc.io guys raised in their video, which is that `isKnownUniquelyReferenced` can only work with pure Swift classes. I presume that this is due to the fact that in Objective-C, non-ARC classes can muck around with `retainCount` or even implement their own reference counting schemes that Swift cannot know about. The way to solve this is to wrap references in a Swift-only wrapper class, which then lets `isKnownUniquelyReferenced` work as expected. If `Foo` subclassed `NSObject` in the previous post's example, the full implementation would have to look like this:

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
  private var _counter = Box(Foo())

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

The first step in abstracting out what we want is to identify the different ways we need to operate on our reference types. There are two general operations we need to perform: mutating data and creating the copy when necessary. This can be done with two closures that have these types:

```swift
/// given a reference to a T, update its contents
typealias Updater<T> = (T) -> Void
/// given a reference to a T, return a new copy of that reference
typealias Copier<T> (T) -> T
```

So let's start constructing a type that can encapsulate this, which we'll call `CopyOnWrite`. First, we need to initialize it:

```swift
struct CopyOnWrite<T: AnyObject> {
  var reference: Box<T>
  private let copy: Copier<T>

  init(reference: T, copy: @escaping Copier<T>) {
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
mutating func update(_ update: Updater<T>) {
  if !isKnownUniquelyReferenced(&_reference) {
    _reference = Box(self.copy(_reference.unbox))
  }

  update(_reference.unbox)
}
```

This will unbox the reference from the swift-only wrapper, pass that along, and box up the result of the copy again transparently so that the caller does not need to worry about doing so themselves ad nauseum. We then call the given update closure to actually update the reference's contents for us.

The function must be `mutating` since we're updating a mutable value in the struct, which will make it impossible to call this function unless the caller is also in a `mutating` function or property (one of the nice ancillary things about value types).

It should be noted here that there's nothing preventing the copy closure from violating the contract of this type by just returning `_reference.unbox` immediately. Enforcing this using the type system isn't possible, so the only option left is to check at runtime if the new reference has an identical pointer as the old one:

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

# Cleaning the API more

The API for mutating an object's state using the `update` method is decent, and using that as a concept to help encapsulate what we're trying to do is helpful. It helps with compartmentalizing the parts of your code that mutate the reference, since it encourages you to wrap up multiple mutating operations inside one closure. However, looking at it a bit more, this might be easier to use if it's wrapped as a separate computed property instead:

```swift
var mutatingReference: T {
  mutating get {
    // copy the reference only if necessary
    if !isKnownUniquelyReferenced(&_reference) {
      _reference = Box(self.copy(_reference.unbox))
    }

    return _reference.unbox
  }
}
```

That would let the Counter's `increment` function look like this:

```swift
mutating func increment() {
  _counter.mutatingReference.count += 1
}
```

...which is a bit easier to type out, especially if you let autocomplete do its thing. If you wanted to do multiple mutating operations in a row it might be more verbose compared to the previously defined `update` method, so choose whichever approach appeals to your sensibilities. We could also just redefine `update` in terms of `mutatingReference` if there's a desire for both ways:

```swift
mutating func update(_ update: Updater) {
    update(mutatingReference)
}
```

Unfortunately, with all of this it's still possible to mutate the reference returned by the normal `reference` parameter, but there's not a good or simple way to guard against that completely. It will be up to the consumer of this API to know when it is appropriate to use `reference` or `mutatingReference` to preserve value semantics for the enclosing type. More advanced tooling might help here, if we could define linter or static analysis rules that detect whether code is referencing things improperly, but that probably couldn't be 100% comprehensive since it's certainly possible for the reference to be accessed and modified dynamically in a way that static analysis couldn't detect.

# Conclusion

Applying all of this to the original `Counter` example yields the following code:

```swift
class Foo: NSObject {
  var count: Int = 0
}

struct Counter {
  private var _counter = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }

  mutating func increment() {
    _counter.mutatingReference.count += 1
    // or this: _counter.update { $0.count += 1 }
  }

  var value: Int {
    return _counter.reference.count
  }
}
```

Much better! The only code that is in here is the actual logic needed to accomplish the task, while the code to handle the copy on write optimization is hidden away within the new struct we made.

There are other opportunities to abstract this out further, which I will go over in a follow-up post, but what we have so far is very usable.
