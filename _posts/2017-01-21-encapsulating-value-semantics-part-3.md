---
title: Encapsulating Value Semantics, Part 3
tags:  swift value-semantics
category: programming
---

In my [last post]({% post_url 2017-01-07-encapsulating-value-semantics-part-2 %}) on this topic, we built up a small struct named `CopyOnWrite` that abstracts away the need to manually check whether or not a reference needs to be copied before modifying it to preserve a type's value semantics. As of last post, the type looks a bit like this (after some small tweaks):

```swift
// Box fits best nested within the `CopyOnWrite` type, but swift 3.0
// unfortunately doesn't support types nested inside generic types.
fileprivate class Box<T> {
  let unbox: T
  init(_ value: T) {
    unbox = value
  }
}

struct CopyOnWrite<T: AnyObject> {

  private var _reference: Box<T>
  private let copy: (T) -> T

  init(reference: T, copy: @escaping (T) -> T) {
    self._reference = Box(reference)
    self.copy = copy
  }

  var reference: T {
    return _reference.unbox
  }

  var mutatingReference: T {
    mutating get {
      // copy the reference only if necessary
      if !isKnownUniquelyReferenced(&_reference) {
        _reference = Box(self.copy(_reference.unbox))
      }

      return _reference.unbox
    }
  }
}
```

What we have so far is pretty nice, but it's still a bit flawed. What if we wanted to add multiple properties of the same type to a struct? We would be forced to provide the same copy closure each time we create an instance of `CopyOnWrite`:

```swift
struct Counter {
  private var _primary = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }
  private var _secondary = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }
  // ...
}
```

Yuck! We have to provide the exact same copy closure each time we create an instance of `Foo`, which isn't good. So, let's extract this copying behavior into its own method on `Foo`:

```swift
extension Foo {
  func clone() -> Self {
    let copy = self.init()
    copy.count = self.count
    return copy
  }
}

struct Counter {
  private var _primary = CopyOnWrite(reference: Foo(), copy: { $0.clone() })
  private var _secondary = CopyOnWrite(reference: Foo(), copy: { $0.clone() })
  // ...
}
```

That's better, but we have to specify the closure that wraps `clone` each time, which still feels like needless repetition. To solve this, we can create a `Cloneable` protocol to represent all copyable types:

```swift
protocol Cloneable: class {
    func clone() -> Self
}
```

(I chose `Cloneable` as a name because I didn't want the method name to conflict with `NSObject`'s `copy()` method.)

We constrain the `Cloneable` protocol to only be applicable to classes, since structs and enums already are copied by default when you assign them to new variables, so it wouldn't make much sense for those types to conform to it.

Now, we can extend `CopyOnWrite` for all cases where `T` conforms to `Cloneable` to provide a new, simpler initializer there. This will make assigning value semantics to a reference in a struct much easier if that reference conforms to `Cloneable`:

```swift
extension CopyOnWrite where T: Cloneable {
    init(reference: T) {
        self.init(reference: reference, copy: { $0.clone() })
    }
}
```

For completeness with existing Foundation types that implement `NSCopying` or `NSMutableCopying`, we can add similar extensions for those protocols:

```swift
extension CopyOnWrite where T: NSCopying {
    init(copyingReference reference: T) {
        self.init(reference: reference, copy: { $0.copy() as! T })
    }
}

extension CopyOnWrite where T: NSMutableCopying {
    init(mutableCopyingReference reference: T) {
        self.init(reference: reference, copy: { $0.mutableCopy() as! T })
    }
}
```

(The init argument names need to be different, otherwise you'll get an ambiguity error when you try to use a type that conforms to more than one of these protocols.)

And now with this, we can finally clean up the original example:

```swift
class Foo: NSObject {
  var count: Int = 0
}

extension Foo: Cloneable {
  func clone() -> Self {
    let copy = self.init()
    copy.count = self.count
    return copy
  }
}

struct Counter {
  private var _primary = CopyOnWrite(reference: Foo())
  private var _secondary = CopyOnWrite(reference: Foo())

  mutating func increment() {
    _primary.mutatingReference.count += 1
  }

  var value: Int {
    return _primary.reference.count
  }
}
```

If we have to add many properties that hold values of the same type, we just have to conform the type to `Cloneable` (or if you like, `NSCopying`/`NSMutableCopying`) and then just wrap the instance in the `CopyOnWrite` struct. That's much easier to read and maintain, isn't it? 😄

# Limitations

I mentioned this previously, but I wanted to revisit it here. While `CopyOnWrite` does some nice things, it doesn't protect you from improper use of the reference type you're wrapping. There's nothing we can leverage within the Swift type system that can stop you from using the non-mutating reference for a mutable operation, nor can it stop you from using the mutating reference when you don't need it (causing needless copies of your referenced data). It's still up to you, the user of this API, to know how the types you're working with behave so that you can use them correctly with this wrapper type.

# Conclusion

We've done a lot to extract all the repetitive boilerplate code into a reusable component that can be applied anywhere you want to have value semantics in your structs, while achieving the performance benefits that come with copy-on-write behavior. I hope that you might find this useful, and if you have any feedback please feel free to contact me on [Twitter](https://twitter.com/kevlario) or via [email](mailto:{{ site.author.email }}). I have a gist of a tweaked version of the final code for `CopyOnWrite` available [here](https://gist.github.com/klundberg/9faf7b934fc2ecbaa8d9fa414905ba7c) as well if you want to use this in your own project.

Until next time!
