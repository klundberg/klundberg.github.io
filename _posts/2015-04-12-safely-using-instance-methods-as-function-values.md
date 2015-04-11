---
title: Safely Using Instance Methods As Function Values
description: "Learn how to avoid unintentional reference cycles when dealing with method values in a functional way."
tags: Swift Functional-Programming
category: programming
hidden: true
---

One of my favorite features of Swift is the fact that functions are first class values: they can be assigned to variables, passed around as arguments to other functions, and manipulated and decorated in numerous ways. You can even do this with methods on classes, structs and enums, since every type of function is represented in the same way in Swift.

When writing swift code that interfaces with classes like `NSNotificationCenter` that have block-based APIs, you can take advantage of this fact by defining a method to respond to notifications, and just pass that method reference along to the notification center.

Consider the following class definition:

{% highlight swift %}
class MemoryCache {

    var observer: AnyObject?

    init() {
        observer = NSNotificationCenter.defaultCenter().addObserverForName(
            UIApplicationDidRecieveMemoryWarningNotification,
            object: nil,
            queue: NSOperationQueue.mainQueue(),
            usingBlock: self.clearMemory)
    }

    deinit {
        NSNotificationCenter.defaultCenter().removeObserver(observer)
    }

    func clearMemory(notification: NSNotification!) {
        // ... do some nontrivial work to empty the cache here
    }

    // ...
}
{% endhighlight %}

Here when we initialize the cache, we listen for memory warning notifications and respond using the given `clearMemory` method. We reference `self.clearMemory` without the parens at the end which just takes the `clearMemory` method as a function value and gives it to the addObserverForName function.

Though this looks nice, there's one very large caveat: when you reference an instance method in this way and store the method value elsewhere, that method strongly references self. If it didn't, your program could crash if the method value outlived the instance that it was bound to. Not crashing is important, but in this scenario it causes a retain cycle, since `MemoryCache` strongly references the notification observer, which strongly references the given closure, which strongly references the `MemoryCache`. At this point we could just cave in and pass a closure into the `usingBlock:` parameter that weakly captures `self`, but that seems a little superfluous here if we only are going to call one other method in that closure anyways (and it's not as fun!)

One valid way you could get around this in some cases by using the target-action pattern from objective-c, but that only works for classes annotated with `@objc` or that inherit from `NSObject` (which our `MemoryCache` does not), and with classes that provide target-action based methods. If we want a more idiomatic Swift solution that is more broadly applicable, we need to think of a better way.

A viable solution needs a way to make the instance method we want to call reference self weakly, and do the right thing when the weak self reference happens to be nil. To do this, we need to take advantage of another neat swift feature: static method references.

**Static method references** are an esoteric feature of Swift types. To get a raw reference to the `clearMemory` method in swift, you can do this:

{% highlight swift %}
let clear = MemoryCache.clearMemory
{% endhighlight %}

Note that we're referencing the clearMemory method as if it were a static/class method and not an instance method. This is valid, but the type of this method may be unexpected:

{% highlight swift %}
(MemoryCache) -> (NSNotification!) -> ()
{% endhighlight %}

This type says that the input argument is an instance of the class in question, and that calling this function like this: `MemoryCache.clearMemory(self)` will return another function with the type `(NSNotification!) -> ()`, which is the type of the actual clearMemory method. Effectively, `self.clearMemory(notification)` is syntactic sugar for `MemoryCache.clearMemory(self)(notification)`.

With all this in mind, we now have a way to create a function using the static method reference of the `clearMemory` method that will be able to weakly capture a reference to self. This function needs to return a function of type `(NSNotification!) -> ()` that we can give to the `addObserverForName` method, and the function we're returning needs to call the staticaly referenced method only if the parameter representing self is not nil. This function looks something like this:

{% highlight swift %}
func weakify(owner: MemoryCache, f: MemoryCache -> NSNotification! -> ()) -> NSNotification! -> () {
    return { [weak owner] notification in
        if let owner = owner {
            f(owner)(notification)
        }
    }
}
{% endhighlight %}

The function we're returning effectively does the same thing as the closure we gave to NSNotificationCenter in the original example. We can't use optional chaining since we've decoupled the object from the method that's being called on that object, but semantically it's equivalent. Now the argument we pass to the usingBlock parameter of the addObserverForName function will look like this:

{% highlight swift %}
weakify(self, MemoryCache.clearMemory)
{% endhighlight %}

However, this `weakify` function is not very reusable; what if we wanted to use this in a completion handler for a UIViewController presentation? As this function currently stands, we can't due to the types it expects. However, we can take advantage of generics here since the exact parameter types don't matter in the context of `weakify`: we're just decorating a function and passing its parameters along. The `weakify` function below works with all methods that take one parameter and return nothing:

{% highlight swift %}
func weakify <T: AnyObject, U>(owner: T, f: T->U->()) -> U -> () {
    return { [weak owner] obj in
        if let owner = owner {
            f(owner)(obj)
        }
    }
}
{% endhighlight %}

`T` will represent the type of the class instance you need to be made weak. It must be constrained to `AnyObject`, since the compiler doesn't let you make weak references to value types like structs/enums (which don't make sense anyways). When used to weakify the `clearMemory` method, `T` will represent the `MemoryCache` and `U` will represent `NSNotification!`.

There are other weakify functions one could write too. A weakify that takes no parameters but returns a value is one interesting case:

{% highlight swift %}
func weakify <T: AnyObject, U>(owner: T, f: T->()->U) -> () -> U? {
    return { [weak owner] in
        if let owner = owner {
            return f(owner)()
        } else {
            return nil
        }
    }
}
{% endhighlight %}

The static method reference will call for a method of the type () -> U, but weakify must return a function of type () -> U?. If the owner is no longer in memory, we cannot call the method reference with it as a parameter, so we must return some other value, and nil is the only reasonable one in this case.

You could even cast values within weakify, for example if you were using a cocoa api that has a block that takes AnyObject as a parameter with a method that expects a more concrete type:

{% highlight swift %}
func weakify <T: AnyObject, U, V>(owner: T, f: T->U?->()) -> V -> () {
    return { [weak owner] obj in
        if let owner = owner {
            f(owner)(obj as? U)
        }
    }
}
{% endhighlight %}

In this one, the method must accept an optional argument, because we're casting to U with the `as?` operator since it is not guaranteed that the type represented by V (the type of the parameter to the block that the original cocoa api would use) can be safely cast to U. You could write a version of this that took a method of type `U->()` instead, but you would need to use the as! operator instead, which runs the risk of crashing your program if the argument types aren't convertible safely.

If you find yourself wanting to use these weakify functions, check out [this gist](https://gist.github.com/klundberg/bf591578ff41f8ad33b3) which has all the weakify functions defined here and them some. If you have any questions, comments, or suggestions, reach out to me via email, twitter, or github. Enjoy!
