---
title: Making 'weakify' better with metatype values
description: "Using metatypes with weakify to achieve more concise code."
tags: swift functional-programming
category: programming
---

My [first major blog post]({% post_url 2015-04-15-capturing-objects-weakly-in-instance-method-references-in-swift %}) here was on crafting the [weakify function](https://gist.github.com/klundberg/bf591578ff41f8ad33b3) and how you could use it to weakly bind a method reference to an instance of the class the method is defined in, so that you could use the resulting function without worrying about potential memory retain cycles. I've used it numerous times in my projects, but in some cases it can get a bit unwieldy, such that an explicit closure would seem to be the shorter option.

In the original post, I defined a class named `MemoryCache`, which when used with `weakify` looks like this:

{% highlight swift %}
let weakClearMemory = weakify(self, MemoryCache.clearMemory)
{% endhighlight %}

That looks pretty good, doesn't it?

However, If you start venturing into the lands of longer class names, you get something more like this (inspired by some recent work I've done):

{% highlight swift %}
let weakHandleSuccess = weakify(self, DynamicContentCollectionViewController.handleSuccess)
{% endhighlight %}

Hrmmm...

This class name is long, but descriptive as to what it does. I wasn't about to shorten it to make it harder to understand elsewhere just to make this line easier to parse. Unfortunately the long class name dwarfs the method name, which is what actually matters.

# Making Things Better

Luckily, we can use metatype values to reference the class without explicitly naming it here. Metatypes are what you actually are dealing with when you call a class method or class property on a class name; the reference to the class name (`MemoryCache` and `DynamicContentCollectionViewController` in my examples above) actually returns a value that represents what effectively is a singleton instance of the metatype of the class. This value can be passed as arguments to functions or stored in variables like any other value.

Objective-C has a way to access the runtime metatype value of an instance by calling the `class` instance method, which returned the most specific class that the instance is a member of. In Swift, every instance of a type (be it class, struct, or enum) can do the same thing with the `dynamicType` property.

Calling `self.dynamicType` will return the same metatype value as referencing the explicit class would (in most cases), and the compiler knows that the type has all the methods defined on the class that represents `self`. Therefore, we can change the example to the following:

{% highlight swift %}
let weakHandleSuccess = weakify(self, self.dynamicType.handleSuccess)
{% endhighlight %}

Much better!

The upshot of this is that this also respects subclasses that override methods. If a subclass of `DynamicContentCollectionViewController` overrides the `handleSuccess` method, and `self` was an instance of that subclass, `self.dynamicType` will properly get the correct class, and therefore the correct instance method reference as well. The original example would have had the wrong behavior since it would have only used the superclass's `handleSuccess` implementation.

For my original example of the `MemoryCache` class, this is strictly more text to type and read; however the correctness of choosing the right method at runtime outweighs this arguable downside. I'd probably be more sympathetic to that argument though if the way to get at an instance's metatype value were something longer like `self.dynamicTypeMetatypeValue` or something else just as absurd. 😁
