---
title: "Fun With Optionals, Part 1 - Basics"
description: "A deep dive into the syntax of declaring optional types and assigning values to optional variables"
tags: Swift Optionals
category: programming
---

Optionals are another one of my favorite features of Swift. When I started learning Objective-C, I was so happy that sending messages to nil was basically a no-op that also returned nil, after coming from Java/C# where null reference exceptions required explicit null checking in most cases. Seeing nullability become a first-class concept in the type system where you can guarantee something will never be nil is the next evolutionary step from that, and I'm glad Swift chose to adopt this from [other languages](https://en.wikipedia.org/wiki/Option_type) that have done so before.

I've been thinking of writing some educational posts on the basic features of swift in addition to more advanced tricks. So, in this series of posts, I'll be going in depth into optionals, how they work, and (eventually) what cool/interesting things you can do with them here that you could not in objective-c or other languages where optionals don't exist.

# The Basics

Optionals are essentially containers that either contain something or nothing, as opposed to the old way of allowing any object type also be able to be nil implicitly. To represent this, `Optional` is its own type which you can find in the standard library:

{% highlight swift %}
public enum Optional<Wrapped> {
    case None
    case Some(Wrapped)
}
{% endhighlight %}

There's more to it than that if you look for yourself, but that's the bare minimum necessary to implement the concept. It's an enum that can either be nothing (the `Optional.None` case) or something (the `Optional.Some(Wrapped)` case, where `Wrapped` stands in for whichever type you want to represent in an optional way). 

## Shorthand Type Definition

Using the above enum, you could declare an optional variable like so:

{% highlight swift %}
var myStr: Optional<String>
{% endhighlight %}

This defines a variable that may either hold a String or nil. Looks good so far, but in a class or struct with many optional instance variables, it might get a little hard to read the actual type you care about with all the `Optional` text strewn about. Luckily Swift provides shorthand syntax to easily declare something as optional (which you may have seen):

{% highlight swift %}
var myOtherStr: String?
{% endhighlight %}

The question mark after the name of the type is the shorthand for `Optional<String>` here. Few other types have this kind of syntactic support, and that's because the concept of optionality is so prevalent that this special syntax makes reading code with optional values a lot easier to read.

## Assigning Values

Using normal enum construction syntax, you'd typically need to specify the cases here whenever you wanted to store a value that may be nil:

{% highlight swift %}
var myOptionalString: String? = .Some("thing")
{% endhighlight %}

This isn't terrible, but it seems superfluous to specify that a value is explicitly non-nil with the `.Some` enum case here. The swift compiler is able to infer that any non-nil value is intended to be the `.Some` case, so you can just type this:

{% highlight swift %}
var myOptionalString: String? = "thing"
{% endhighlight %}

That's better! This is also known as auto-wrapping or auto-boxing of the value into an optional value of the same type. It helps developers use optionals more easily for the same reason we have special type syntax: to encourage optional adoption by eliminating unneeded boilerplate. You can still be explicit by using enum construction, but it adds no extra information of the context or intent of the code, so it's best to just let the compiler handle that for you.

## The Actual `nil` literal

By the same vein, assigning a nil value to a variable normally would have to look like this if `Optional` didn't have specific syntactic support:

{% highlight swift %}
var myOptionalString: String? = .None
{% endhighlight %}

Since we're able to eliminate `.Some` with automatic boxing of values into optionals, we should be able to do the same with .None so that the fact that it's an actual enum does not need to be known. This is where `nil` finally comes in. You can rewrite that line to look like this instead

{% highlight swift %}
var myOptionalString: String? = nil
{% endhighlight %}

40% shorter and just as expressive. The reason this works is that `Optional` also conforms to a protocol called `NilLiteralConvertible`. This protocol defines a specific init method that must be implemented to create what represents a `nil` value for whichever type conforms to that protocol. The full protocol definition from the Swift standard library is this:

{% highlight swift %}
public protocol NilLiteralConvertible {
    public init(nilLiteral: ())
}
{% endhighlight %}

The compiler knows about `NilLiteralConvertible` and converts all references to `nil` to the above `init(nilLiteral:)` initializer in order to ultimately get the correct value, which for  any optional is just `.None`. For `Optional` to satisfy this protocol, all it needs to do is this:

{% highlight swift %}
public init(nilLiteral: ()) {
    self = .None
}
{% endhighlight %}

Pretty neat!

#Conclusion

We've only scratched the surface here of how optionals work. Swift's native syntax and compiler support for optionals makes it as easy to assign values to an optional type as assigning to a normal typed variable, with auto-wrapping and the nil literal conversion at the compiler level.

Next time I'll expand on all of this by going into how to actually get values out of optionals and operate on them concisely and safely. Until then!