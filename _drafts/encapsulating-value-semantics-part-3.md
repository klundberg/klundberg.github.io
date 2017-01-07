---
title: Encapsulating Value Semantics, Part 3
tags:  swift value-semantics
category: programming
---

What we have so far is pretty nice, but it's still a bit flawed. What if we wanted to add multiple properties of the same type to a struct? We would be forced to provide the same copy closure each time we create an instance of `CopyOnWrite`:

```swift
struct Counter {
  private var first = CopyOnWrite(reference: Foo()) { old in
    let copy = Foo()
    copy.count = old.count
    return copy
  }
  private var second = CopyOnWrite(reference: Foo()) { old in
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
  private var first = CopyOnWrite(reference: Foo(), copy: Foo.init(copying:))
  private var second = CopyOnWrite(reference: Foo(), copy: Foo.init(copying:))
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
  private var _primary = CopyOnWrite(reference: Foo())
  private var _secondary = CopyOnWrite(reference: Foo())

  mutating func increment() {
    _primary.update { $0.count += 1 }
  }

  var value: Int {
    return _primary.reference.count
  }
}
```

If we have to add many properties that hold values of the same type, we just have to conform the type to `Copyable`, wrap all construction or assignment of those references into the `CopyOnWrite` struct, and that's basically it!

# Conclusion

We've done a lot to extract all the repetitive boilerplate code into a reusable component that can be applied anywhere you want to have value semantics in your structs, while achieving the performance benefits that come with copy-on-write behavior. I hope that you might find this useful, and if you have any feedback please feel free to contact me on [Twitter](https://twitter.com/kevlario) or via [email](mailto:{{ site.author.email }}). I have a gist of a tweaked version of the final code for `CopyOnWrite` available [here](https://gist.github.com/klundberg/9faf7b934fc2ecbaa8d9fa414905ba7c) as well if you want to use this in your own project.

Until next time!
