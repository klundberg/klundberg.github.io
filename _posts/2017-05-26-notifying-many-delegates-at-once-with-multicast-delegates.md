---
title: "Notifying many delegates at once with Multicast Delegates"
tags: swift objective-c delegate-pattern
category: programming
---

At work, we're investigating new architectures to use in our app that should make it easier to develop and test our code across the many teams who work on our codebase. We've taken a look at the Coordinator pattern and VIPER, and in both scenarios we've run into cases where we want multiple objects to have the ability to listen to delegate callbacks from a UINavigationController, to help make cleaning up view controllers on a `UINavigationController` stack easier.

Soroush Khanlou recently [blogged about this](http://khanlou.com/2017/05/back-buttons-and-coordinators/), but his preferred solution appears to not take into account the case where you might want multiple coordinators to manage view controllers on the same UINavigationController. In this case, you would need to implement something called a multicast delegate, which can distribute delegate callbacks among every coordinator/router that should care about them.

It's important to note that implementing this concept is very different in Objective-C when compared to swift, due to the nature of how each language works and what capabilities they expose. I'll go over what an implementation could look like in each, and you can decide which version you may prefer. To keep things simple, I'm going to ignore concerns like thread safety, but operating on these multicast delegates might require some thought if you intend to modify them in multiple threads.

## Objective-C implementation

Most of the time, you are told to subclass `NSObject` when creating a new class type in Objective-C, since that root class has all of the memory management/dynamic method invocation stuff you need to make things work. However, there's another root class that has some interesting uses that is perfect for this, named `NSProxy`. This must be subclassed in in Objective-C, because in Swift the methods you need to override expose types that are unavailable there, namely `NSInvocation` and `NSMethodSignature`.

First off, we need a way to store references weakly in a collection. There are numerous ways to do this, some using `NSPointerArray`, `NSMapTable`, or using Core Foundation to build arrays with specific object lifetime behaviors, but for simplicity's sake (and ease of comparing with Swift's version) I'm going with a simple wrapper class:

```
@import Foundation;

@interface OCWeakRef : NSObject
@property (nonatomic, weak, nullable, readonly) id value;
+ (nonnull instancetype)weakRefWithValue:(nullable id)value;
@end

@implementation OCWeakRef
- (instancetype)initWithValue:(id)value {
    self = [super init];
    if (self) {
        _value = value;
    }
    return self;
}

+ (instancetype)weakRefWithValue:(id)value {
    return [[self alloc] initWithValue:value];
}

- (BOOL)isEqual:(id)object {
    return [object isKindOfClass:[WeakRef class]] && [[(WeakRef *)object value] isEqual:self.value];
}

- (NSUInteger)hash {
    return [self.value hash];
}
@end
```

This will store the object weakly in the `value` property and also participate properly in collection behavior with simple implementations of `isEqual:` and `hash`. Now, we can actually build our delegate:

```
@import Foundation;

@interface OCMulticastDelegate: NSProxy
@property (nonatomic, copy, readonly, nonnull) NSMutableArray<WeakRef *> *delegates;
- (nonnull instancetype)init;
- (void)addDelegate:(nonnull id)delegate;
- (void)removeDelegate:(nonnull id)delegate;
@end

@implementation OCMulticastDelegate
- (instancetype)init {
    _delegates = [NSMutableArray array];
    return self; // NSProxy does not implement an init method, so we cannot call [super init]
}

- (void)addDelegate:(id)delegate {
    [self.delegates addObject:[OCWeakRef weakRefWithValue:delegate]];
}

- (void)removeDelegate:(id)delegate {
    [self.delegates removeObject:[OCWeakRef weakRefWithValue:delegate]];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector {
    [self clearNilDelegates];
    return [[self.delegates.firstObject value] methodSignatureForSelector:selector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    for (OCWeakRef * delegate in self.delegates) {
        if (delegate.value != nil) {
            [invocation invokeWithTarget:delegate.value];
        }
    }
}

- (void)clearNilDelegates {
    for (OCWeakRef * delegate in self.delegates) {
        if (delegate.value == nil) {
            [self.delegates removeObject:delegate];
        }
    }
}
@end
```

Here we have a pretty simple looking class that can build up a set of child delegates for the `MulticastDelegate` to notify. When any method is called on the multicast delegate, `methodSignatureForSelector:` and then `forwardInvocation:` are called to resolve what should happen as a result (pruning any nilled delegates in the process).

The `methodSignatureForSelector:` call is just passed along to the first item in the delegates list, since we need to choose one, and ideally all these delegates should conform to the same protocol. The result of that call is then used to build up the `NSInvocation` that's passed into the `forwardInvocation:` method. Since we want to call any method on the child delegates, we simply loop over them and run the invocation on each one.

Now, you may be asking how you would make this object conform to your desired protocol. You can make that happen by doing the following:

```
@import UIKit;

@interface OCMulticastUINavigationControllerDelegate: OCMulticastDelegate <UINavigationControllerDelegate>
@end

@implementation OCMulticastUINavigationControllerDelegate
@end
```

For the simple cases where your delegate methods don't have a return value, that's it! Since the `NSProxy` automatically resolves methods using `methodSignatureForSelector:` and `forwardInvocation:`, calling any of the supported methods here will automatically forward them on to the child delegates for you. Providing the generic parameter as the delegate you want, and then saying you conform to it should be enough to give you some compile time hints to make sure you're adding the right delegates as children.

If your delegate methods do return something, you need to take more care. Leaving it to the superclass will simply leave you with the return value of the last delegate that was called, which may not be what you want. You may want to combine all the results somehow, or only return the first result instead of the last. If you require that, then you'll have to manually implement each method that has these requirements in the specialized subclass. If you do this, then that method will be called directly rather than `methodSignatureForSelector:` and `forwardInvocation:`, making sure this has the effect you want.

### Caveats

This looks nice and compact considering that it's Objective-C, since it's taking advantage of the robust runtime behavior that Objective-C provides. However it has some drawbacks, especially if you're using this from Swift.

 1. You can't enforce the types of child delegates added to the multicast delegate here. It's very easy to accidentally add something that doesn't conform to `UINavigationControllerDelegate` in the above example, and you'd get a runtime error whenever something tried to call one of that delegate's methods on the non-conforming object.
 2. Since the runtime is responsible for handling method calls that aren't implemented on the delegate directly, you cannot get any compiler feedback if you need to implement a method explicitly for your use case, you just have to test at runtime to figure it out.
 3. This only works on Apple platforms, since it relies on the Objective-C runtime and `NSProxy`. You can't use this on linux or any other place Swift may run.

## Swift implementation

If you only need to use this from Swift, we can write a more type-safe version in that language. Such an implementation may look like this:

```swift
fileprivate final class Weak<Value: AnyObject> {
    weak var value: Value?
    init(_ value: Value) {
        self.value = value
    }
}

// Subclass NSObject so that obj-c protocols can conform to this on Apple platforms.
class MulticastDelegate<T: AnyObject>: NSObject {
    fileprivate var delegates: [Weak<T>] = []

    public override init() {
        super.init()
    }

    func addDelegate(_ delegate: T) {
        self.delegates.append(Weak(delegate))
    }

    func removeDelegate(_ delegate: T) {
        guard let index = delegates.index(where: { $0.value === delegate }) else {
            return
        }
        self.delegates.remove(at: index)
    }

    fileprivate func clearNilDelegates() {
        for (index, element) in delegates.enumerated().reversed() {
            if element.value == nil {
                delegates.remove(at: index)
            }
        }
    }
}

extension MulticastDelegate: Sequence {
    func makeIterator() -> AnyIterator<T> {
        clearNilDelegates()

        var iterator = delegates.makeIterator()

        return AnyIterator {
            // loop over the generator's items until we find one that isn't nilled out
            //  and return that as the next item in this generator.
            // Even though we preemptively remove nil delegates before starting,
            //  it's possible something else could cause a delegate to become nil during iteration.
            while let next = iterator.next() {
                if let delegate = next.value {
                    return delegate
                }
            }
            return nil
        }
    }
}
```

This implementation is certainly a lot longer than the Objective-C version, and it doesn't appear to do as much. It provides a way to weakly hold onto a list of delegates, lets the user add/remove them, and lets consumers of this API iterate over the delegates to use them without worrying about the `Weak` wrapper, since we conform to  `Sequence`. With this in place, we can subclass this and conform to whatever delegate we care about.

```swift
class MulticastUINavigationControllerDelegate: MulticastDelegate<UINavigationControllerDelegate>, UINavigationControllerDelegate {

  public override init() {
    super.init()
  }

  @objc open func navigationController(_ navigationController: UINavigationController, willShow viewController: UIViewController, animated: Bool) {
    for delegate in self {
      delegate.navigationController?(navigationController, willShow: viewController, animated: animated)
    }
  }

  // ...
```

We have to manually implement all the delegate methods on the protocol but everything we've done in the superclass makes implementing these methods very easy. In this case all we need to do is loop over everything in self and call the method we're currently implementing on every child delegate.

In the case of `UINavigationControllerDelegate`, many of the methods return something, so we will need to handle those cases somehow. For `navigationControllerSupportedInterfaceOrientations(_:)`, we need to determine which interface orientations should be supported. Since this returns an OptionSet, the most reasonable thing to do would be to take every child delegate that implements this and get the intersection of all their results, like so:

```swift
@objc open func navigationControllerSupportedInterfaceOrientations(_ navigationController: UINavigationController) -> UIInterfaceOrientationMask {
  // combine all results from every delegate and find the common supported orientations among them all.
  return reduce(.all, { (mask, delegate) in
    guard let newMask = delegate.navigationControllerSupportedInterfaceOrientations?(navigationController) else {
      return mask
    }
    return mask.intersection(newMask)
  })
}
```

For a method like `navigationControllerPreferredInterfaceOrientationForPresentation(_:)`, if you treat your multicast delegate like a stack, it makes the most sense to have the last child control the preferred orientation, since it would be responsible for the topmost view controllers on the navigation stack. If no children implement the method, we need some other reasonable default, so we defer to the window's current orientation, or just portrait if for some reason the navigation controller is not attached to a window:

```swift
@objc open func navigationControllerPreferredInterfaceOrientationForPresentation(_ navigationController: UINavigationController) -> UIInterfaceOrientation {
  for delegate in self.reversed() {
    if let orientation = delegate.navigationControllerPreferredInterfaceOrientationForPresentation?(navigationController) {
      return orientation
    }
  }
  // if not implemented by any delegate, use a reasonable default
  return navigationController.view.window?.screen.interfaceOrientation ?? UIInterfaceOrientation.portrait
}
```

Methods like the view controller transitioning methods would follow the same logic as above, since the last delegate added would likely manage the transition style of the topmost view controllers on the nav stack.

### Caveats

1. Obviously this is a little bit more code to write compared to Objective-C, since Swift lacks the runtime features that Objective-C has to make this less verbose for the simple cases. You will always be forced to implement all non-optional protocol methods on your `MulticastDelegate` subclass, which means you will have to think more carefully about how you want your multicast implementation to behave.
2. To make sure you don't ignore a child delegate's optional method implementation, you will need to implement all optional methods as well, and dispatch to the right child delegate(s) as appropriate. This however may yield different behavior than if you didn't implement the method on the multicast, since many Cocoa classes do something specific if the delegate method isn't implemented on the delegate. It's up to you to read the documentation and implement that same default behavior as best you can, and even so it may be impossible to fully get back the behavior that happens when a method is unimplemented.

## Conclusion

Each implementation has its downsides and upsides, but my preferred implementation is the Swift version, since it provides stronger guarantees on the types the multicast delegate may contain, and it forces you to think more carefully about how you implement dispatching to all the child delegates. The extra required boilerplate code that the Objective-C version obviates can likely be generated for you with a code generation tool if you really want to avoid specifying things manually.
