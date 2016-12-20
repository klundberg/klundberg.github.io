---
title: Encapsulating Value Semantics, Part 1
tags:  Swift value-semantics
category: programming
---

I watched a couple of really good videos on value semantics recently. One was by Alexis Gallagher, named [Value SEMANTICS (not value types!)](https://realm.io/news/swift-gallagher-value-semantics/), which does a good job explaining the differences among value types, reference types, value semantics and reference semantics. Another is by Chris and Florian over at objc.io, named [Understanding Value Type Performance](https://talk.objc.io/episodes/S01E20-understanding-value-type-performance) where they demonstrate how to wrap a class in a struct and ensure that this new type has value semantics (this one is a subscriber-only episode, but subscribing to their series is worth it). Both videos are really good, and I highly recommend watching them (or at least reading the transcripts).

In each video as an aside, the idea of encapsulating the code behind delivering these value semantics is put forward. I thought that this would be an interesting exercise, so I decided to explore how that would look. In this post I'll do a short intro/refresher to how value semantics work (if you don't have time to watch those videos), and I'll explore how to encapsulate that behavior in one easy to use type in part 2 of this series.

# So, what are value semantics?

Value semantics are effectively what you get when the only way you can modify a data structure is by modifying the variable it's stored in. This means that any type that has these semantics cannot be affected outside of the scope of the variable that holds that value. Alexis's talk describes a way to test this behavior that's called *"The Mutation Game"*, which effectively boils down to this:

```swift
var original = Foo()
var copy = original
// make changes to `copy` to try to affect `original` without touching `original` directly
assert(copy != original)
```

If any code is able to change `copy` without touching the `original` variable directly, then the type of those variables does not have value semantics since the changes aren't contained only to `copy`. However if any possible change to `copy` only affects the value in the `copy` variable, then that type does have value semantics.

# Value semantics, how do they work?

Value types like `struct`s and `enum`s in the basic case can have value semantics. You cannot affect one `Array` by appending to another for instance (as `Array` is a struct):

```swift
var a = [1,2,3]
var b = a
a.append(4)
print(a) // [1,2,3,4]
print(b) // [1,2,3]
```

The same behavior is seen in things like CGSize with property access:

```swift
var s = CGSize(width: 10, height: 100)
var s2 = s
s.height = 10
// s2.height is still 100
```

Modifying one value has no effect on the other one, which makes reasoning about what the value of something could be during your code's execution much easier.

However, when you introduce classes (which are reference types) into the picture, things get a little muddy:

```swift
class Foo {
  var count: Int = 0
}

struct Counter {
  var _counter = Foo()

  // doesn't have to be mutating since `_counter` is a class, and making changes
  // to a class's properties doesn't change the class value itself
  func increment() {
    _counter.count += 1
  }

  var value: Int {
    return _counter.count
  }
}

var c = Counter()
let c2 = c
c2.increment()
assert(c.value != c2.value) // will trap!
```

In this (contrived) example, since `Foo` is a class, only the reference stored in the struct is copied to the new location. Those references still point to the same instance of `Foo` in memory, so changing one `Foo` reference's internals will make that change visible to both `Counter` values.

# OK, so how do I make value semantics work when my structs contain classes?

If you want your structs to have value semantics in this case (where your data is stored in a class and is mutable), you need to copy the class instances whenever something would make a change to that data:

```swift
struct Counter {
  var _counter = Foo()

  // has to be mutating now, since we're actually assigning a new reference to
  // the `_counter` variable
  mutating func increment() {
    // copy the reference whenever we increment, so that we know only the value
    // that calls this method will get updated
    var old = _counter
    _counter = Foo()
    _counter.count = old.count

    _counter.count += 1
  }

  var value: Int {
    return _counter.count
  }
}

var c = Counter()
let c2 = c
c.increment()
c.increment()
assert(c.value != c2.value) // will pass!
```

This will work, but it's not optimal. Notice how I called `increment` twice in this example. Here, we will allocate an instance of `Foo` each time `increment` is called, but we don't need to do that after the first time since `c`'s data is unique to it after the first `increment` call, making the second allocation worthless. You can solve this by implementing a behavior called "copy on write", which means that any reference types are only copied when you actually need to mutate them.

This is possible using a function from the standard library named `isKnownUniquelyReferenced`:

```swift
struct Counter {
  var _counter = Foo()

  mutating func increment() {
    // copy the reference only if necessary
    if !isKnownUniquelyReferenced(&_counter) {
      var old = _counter
      _counter = Foo()
      _counter.count = old.count
    }

    _counter.count += 1
  }

  var value: Int {
    return _counter.count
  }
}

var c = Counter()
let c2 = c
c.increment()
c.increment() // won't allocate!
assert(c.value != c2.value)
```

`isKnownUniquelyReferenced` will check how many references exist to the given object, and if it returns false, we need to make a copy of the object so that we don't affect other values that reference the original object.

Now, if you call increment two or more times in a row, you'll no longer get a performance penalty after the first increment due to extra memory allocations. And if you never even assigned to `c2` in the first place, no allocations would occur at all when calling `increment` since there's only one reference to `Foo` alive here. This copy on write behavior is an optimization that's used in the standard library and Foundation, with types like `String`, `Array`, `Data` and others in order to deliver the best performance they can, and you should do the same for your programs that might need this behavior.

# So what's next?

I should point out that you can get value semantics more easily in other scenarios. If your structs only contain other types that already have value semantics, you don't necessarily have to do anything like this. You could also consider a class that doesn't let you mutate its state after initialization to have value semantics, since immutable values by definition cannot be changed without actually reassigning to the variable that contains it. This technique is only needed when you want to edit data held by references in your structs.

Even so, you can also probably already imagine that doing this check often could get unwieldy. If you need to mutate a reference type in multiple places, you'd need to do call `isKnownUniquelyReferenced` each time, and repeating that incantation opens the door to typos and subtle bugs. You can move that into a property or function within your struct, but this is a pattern that will crop up often if you want value semantics in many types, so a more comprehensive solution could be nice. I'll go into that in the next post on this subject. Until next time!
