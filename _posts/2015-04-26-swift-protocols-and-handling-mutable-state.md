---
title: Swift Protocols And Handling Mutable State
description: "The nuances of swift protocols and how they handle object and value types."
tags: swift
category: programming
---

I recently ran into a curious problem on one of my side projects that I didn't understand at first. I was using a swift protocol to provide a uniform interface for various possible kinds of collection view cells that all show a common source of data in different ways, and have a single delegate that they communicate through to respond to user input. A distilled version of what I was doing looked something like this:

{% highlight swift %}
protocol MyCellType {
    var mydelegate: MyCellDelegate? { get set } // another swift protocol
    func configure(model: MyModel)
}

// later, in a cellForItemAtIndexPath call:

let cell = collectionView.dequeueReusableCellWithReuseIdentifier("cell", forIndexPath:path) as! UICollectionViewCell
if let mycell = cell as? MyCellType {
    mycell.mydelegate = mydelegate // pass along the delegate owned my this class into each cell
    mycell.configure(model)
}
{% endhighlight %}

It may not be guaranteed that all cells served by the collection view will conform to this protocol, so using optional binding with if-let is the natural way to handle this case for when the cells are the type I expect.

> (As a side note, it's frustrating that this method on UICollectionView returns `AnyObject!`  instead of `UICollectionViewCell?`. I use `as!` here as I cannot fathom a time when this method doesn't return a collection view cell; if there is some situation please let me know!)

When I initially wrote this code, I was confused to see a warning show up on the first line inside the if-let statement:

> Cannot assign to 'mydelegate' in 'mycell'

Why not? It didn't make sense to me. I figured that since the `MyCellType` protocol was applying to an `@objc` type (the `UICollectionViewCell`) that the protocol itself also needed to be `@objc`:

{% highlight swift %}
@objc protocol MyCellType {
    var mydelegate: MyCellDelegate? { get set } // another swift protocol
    func configure(model: MyModel)
}
{% endhighlight %}

This solved my problem, but yielded another. The `MyCellDelegate` protocol was not an `@objc` protocol, so the compiler complained that I needed to change that protocol as well. It's certainly reasonable that `@objc` protocols can't reference non `@objc` protocols inside of them. I wasn't terribly happy with this though since none of the code that was using these protocols was written in Objective-C (even though it's a mixed-language app). If I wanted to use tuples, structs, or generics in these protocols later on then I'd be out of luck.

After playing with this a bit in a playground I was luckily able to stumble onto the answer. The bare essence of the problem can be expressed with the following code:

{% highlight swift %}
protocol MyType {
    var name: String { get set }
}

class MyThing: MyType {
    var name = "blah"
}

let thing1 = MyThing()
thing1.name = "Thing 1"
let thing2: MyType = MyThing()
thing2.name = "Thing 2"
{% endhighlight %}

Setting the name on `thing1` works fine, but setting it on `thing2` gives us the same compiler issue unless the protocol is `@objc`. Note that we're not subclassing any `@objc` classes so that's clearly not the issue here.

I tried changing `MyThing` to be a struct to see if the behavior was different, and the following happened:

![Cannot assign to 'name' in 'thing1'. Cannot assign to 'name' in 'thing2'](/images/protocol-1.png)

Which makes sense, since structs assigned to constants are fully immutable: the reference to the struct and the members of the struct are all unable to be changed. Changing the `let` to `var` for each fixed the errors.

This led me to the eureka moment. The Swift type system cannot reasonably know that the value in a constant whose type is a plain protocol is in fact a class type. If `MyThing` really was meant to be a struct, then modifying those fields in the constant definitely should not be allowed. If I'd had any functions on the `MyType` protocol that were tagged as `mutating`, then trying to call those on `thing2` would have yielded a similar error. If I change `MyThing` back to a class and leave `thing2` as a `var`, everything works great.

Since `var` works for `MyThing`, we could also change the original collection view cell example to use if-var instead of if-let to make things work:

{% highlight swift %}
if var mycell = cell as? MyCellType {
    mycell.mydelegate = mydelegate // pass along the delegate owned my this class into each cell
    mycell.configure(model)
}
{% endhighlight %}

Doing this lets us remove the `@objc` attributes from `MyCellType` and `MyCellDelegate`.

However, I prefer to use `let` wherever possible and use `var` only when necessary, and in this case, there's no semantic need for those values to be `var` since their references don't change (though their contents do).  The reason `@objc` fixed this and allowed my code to use `let` is not because the class was also an Objective-C class, but because that attribute also constrains protocols to only apply to class types (which is a swift feature I had forgotten about). We can change the original `MyCellType` protocol's definition to only allow it to apply to class types by doing the following:

{% highlight swift %}
protocol MyCellType: class {
    var mydelegate: MyCellDelegate? { get set } // another swift protocol
    func configure(model: MyModel)
}
{% endhighlight %}

This will also have the desired effect, without restricting the kinds of swift features we can use within this protocol. The type system can know that editing a property of a constant of this type is allowed since it can't be applied to value types, and therefore we don't need to store a `MyCellType` in a var to edit its contents.

It's certainly a good thing that the compiler yelled at me for doing this the wrong way originally, as if it hadn't complained and I had used `MyCellType` with a struct later on, I could have violated swift's fundamental invariant for value types stored in a constant. However, I know I would not have done this, and telling the compiler about my intentions with a class protocol makes both me and the compiler happy.

Putting the proper thought into designing your protocols is crucial to getting your code to behave the way you want, and that includes considering if your protocol is meant to behave in a class-like way or in a much broader way. With this distinction, the language can know more about your intentions with how you choose to express your ideas in code, and it can provide benefits like class-based property access along the way.
