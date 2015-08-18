---
title: Swift 2 and @available properties
description: How the @available annotation in swift 2 may get in your way.
tags: Swift 
category: programming
---

I haven't played around with Swift 2 too much yet as I've been busy with other things in the past couple of months. However today I fired up Xcode 7 (beta 4) and tried converting the app I'm working on at work to Swift 2. It's a hybrid app with much more Obj-C code than swift code presently, so there wasn't too much for the converter to do, only a few syntactical changes in 8-10 files or so.

However, once I tried building the app, one of the classes that wasn't a problem during conversion started throwing compiler errors. An excerpt of the problem code looks like this:

{% highlight swift %}
class ViewController: UIViewController {
    var searchController: UISearchController?
    // error: 'UISearchController' is only available on iOS 8.0 or newer
}
{% endhighlight %}

This is mildly annoying but not unexpected, as our app runs on iOS 7 and up. Swift 2 is stricter about API usage, but it gives developers a tool to address this problem.

@available/#available
-

Swift 2 has a compiler-friendly way of checking whether or not you're using APIs that are only available on a specific version of one of apple's operating systems, with the `@available` attribute and the `#available` conditional check. If you are using a class like `UISearchController` on an app that can run on an OS where that class doesn't yet exist, you need to check for it. At runtime, you'd do so like this:

{% highlight swift %}
func makeSearchController() {
    if #available(iOS 8.0, *) {
        self.searchController = UISearchController()
    } else {
        // do something appropriate when on iOS 7
    }
}
{% endhighlight %}

When using `#available`, the compiler can know at compile time that the code within the `#available(iOS 8.0, *)` check can only run if the OS is iOS 8 or above, so it allows you to use APIs that only work on that iOS version. Code within the else block runs if the `#available` check fails, i.e. if the user is running on iOS 7 here. This is all in the name of safety, as using classes that aren't available on some of your supported OS versions can cause unexpected crashes.

What if we wanted to make a function that returns some type that is only available on iOS 8 and above though?

{% highlight swift %}
func makeAnotherSearchController() -> UISearchController {
    return UISearchController()
}
{% endhighlight %}

We can't use runtime checks with `#available` here since the offending class is in the type signature, and what would we return from here on iOS 7 anyways even if we could? We instead have to use the `@available` attribute to annotate the function's intent that it only be callable on iOS 8 and above:

{% highlight swift %}
@available(iOS 8.0, *)
func makeAnotherSearchController() -> UISearchController {
    return UISearchController()
}
{% endhighlight %}

This compiles just fine, and allows the function to refer to types/methods only in iOS 8 without a runtime `#available` check anywhere within the method. However, any method calling this function would need to wrap it in an `#available` check since this method itself has the same availability as UISearchController does.

You can use `@available` on methods, properties and classes. A class marked as `@available` for a specific version will have that version applied to all its members, so you won't need to check for `#available` for that version at all within the class, which is pretty nice. The full documentation on how both work can be found in the prerelease versions of [Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/AdoptingCocoaDesignPatterns.html#//apple_ref/doc/uid/TP40014216-CH7-ID6) and [The Swift Programming Language](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Attributes.html#//apple_ref/doc/uid/TP40014097-CH35-ID347).


The Problem At Hand
-

Now, armed with this knowlege, it must be a simple matter of adding `@available` to my class's property to make this work, right?

{% highlight swift %}
class ViewController: UIViewController {
    @available(iOS 8.0, *)
    var searchController: UISearchController?
    // error: Stored properties cannot be marked potentially unavailable with 'introduced='
}
{% endhighlight %}

Gah! Apparently `@available` works almost everywhere, with stored properties being an exception. It makes some sense, as what would this property store on iOS 7? Since it's optional, it could just be forced to be nil, but what if it's not optional? I presume that the compiler cannot reason about what to do in this case, so it just complains if you try.

One solution here would be to annotate the whole class as being available only in iOS 8+, but that's not acceptable in my case, as the class where this property was defined still must be accessible on iOS 7, at least until we drop support altogether for that OS version. Unfortunately the only solution I could come up with is this ugly workaround:

{% highlight swift %}
class ViewController: UIViewController {

    // Yuck
    private var _searchController: AnyObject?
    @available(iOS 8.0, *)
    var searchController: UISearchController? {
        get {
            return _searchController as? UISearchController
        }
        set {
            _searchController = newValue
        }
    }
}
{% endhighlight %}

The compiler lets this through, since computed properties don't change the memory layout of the object, so the object won't have any direct references to a class that may not exist on some devices. It's still a hassle to do though when you can't make your types use the same availability restriction.

I was encouraged to file a bug in #swift-lang on freenode about this, which i've also mirrored on [openradar](https://openradar.appspot.com/radar?id=5683242493542400) with some potential implementation solutions (TL;DR: make `@available` stored properties be optional in case they cannot be set on older OS versions). Hopefully Apple will take a liking to that suggestion!

