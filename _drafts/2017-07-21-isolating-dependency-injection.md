---
title: "Isolating Dependency Injection"
tags: swift dependency-injection
category: programming
---

Dependency injection (DI) is a valuable technique to make your code more testable and less coupled. With it, you can eliminate singletons from your code and make it painless to substitute the implementation of one dependency with another, such as replacing something with a mocked variant in a unit test.

The concept of dependency injection has been written about a great deal, so I'll refer you to some other sources that go into greater depth (TODO). Suffice it to say that in a very complex application, you may be considering using a dependency injection framework to handle wiring up everything together. One such framework that I like is Swinject, which has lots of features but keeps your dependency configuration no more complex than you need it to be.

However, introducing such a framework can tightly couple your application to that framework, and should you decide to do things differently and go with another framework or resolve your dependencies manually, refactoring everything could take a lot of effort. My goal here is to show how we can structure our code to take advantage of a third party framework while also limiting that framework's exposure to the rest of our code, making replacing that framework easy.

First off, to illustrate how this sort of framework can help, I'll use this set of types as an example (note that it's just the shape of the types, I'll leave any implementation to your imagination):

```swift
class APIService {
  struct Environment {
    let baseURL: URL
  }

  let urlSession: URLSession
  let deserializer: ResponseDeserializer
  var environment: Environment
}

class ResponseDeserializer {
  let queue: DispatchQueue
}

class ListViewController {
  let apiService: APIService
}

class DetailViewController {
  let apiService: APIService
}
```

Here we have some view controllers that depend on an `APIService`, which in turn depends on an `Environment`, a `URLSession` and an object responsible for deserializing responses into models (which also depends on a dispatch queue to put its work on). Now how do we connect all of these dependencies? Well we could do it imperatively with constructor dependency-injection:

```swift
let deserializer = ResponseDeserializer(queue: DispatchQueue.global())
let environment = Environment(baseURL: myURL)
let service = APIService(urlSession: URLSession.shared, deserializer: deserializer, environment: environment)
let controller = ListViewController(service: service)
```

This however leaks knowledge about everything in the dependency chain. You need to know what one component takes and obtain or create that somehow, before creating the next component which requires it. You could set up something like the `APIService` ahead of time and share that reference around, but if you present a set of view controllers one after another, view controllers which don't need an `APIService` will have to hold a reference to it just to have all the requirements for the next one that does need a reference.

This is where a DI framework can help. You can register all your dependencies in a dependency container, and resolve them when you need them using that same container. This will let you pass around one object that has access to all the dependencies your system may need, and each part of the system. Using Swinject, we can create what is called a container for our dependencies and tell the container how to create them when we ask for one:

```swift
// called somewhere early in the app lifecycle, like the app delegate:
func registerDependencies() -> Container {
  let container = Container()
  container.register(DispatchQueue.self, factory: { _ in DispatchQueue.global() })
  container.register(ResponseDeserializer.self, factory: { r in
    return ResponseDeserializer(queue: r.resolve(DispatchQueue.self)!)
  })
  container.register(Environment.self, factory: { _ in
    return Environment(baseURL: URL(string: "https://example.com")!)
  })

  container.register(URLSession.self, factory: { _ in URLSession.shared })
  container.register(APIService.self, factory: { r in
    return APIService(urlSession: r.resolve(URLSession.self)!,
                      deserializer: r.resolve(ResponseDeserializer.self)!,
                      environment: r.resolve(Environment.self)!)
  })

  container.register(ListViewController.self, factory: { r in
    return ListViewController(apiService: r.resolve(APIService.self)!, resolver: r)
  })

  container.register(DetailViewController.self, factory { r in
    return DetailViewController(apiService: r.resolve(APIService.self)!)
  })

  return container
}
```

This looks pretty verbose for setting dependencies up, but its pretty powerful in that each dependency registration does not have to care much at all about how its own dependencies are created, it just asks the dependency resolver it gets to resolve what it needs in order to be created, and asking for something else is just a matter of asking the resolver to resolve a type differently, or with some extra context (for instance you can register multiple instances of the same type and give each one a unique name, so you can choose which one to grab).

Now all we need to do is make the view controllers use these containers to get what they need:

```swift
class ListViewController {
  let apiService: APIService
  private resolver: Resolver // in Swinject, Container conforms to Resolver and we only get an abstract Resolver back when resolving dependencies

  init(resolver: Resolver) {
    self.resolver = resolver
    apiService = resolver.resolve(APIService.self)!
  }

  func showNext() {
    present(resolver.resolve(DetailViewController.self))
  }
}

class DetailViewController {
  let apiService: APIService

  init(container: Container) {
    apiService = container.resolve(APIService.self)!
  }
}
```

This is pretty good, but as you can see the view controllers now have to know that dependencies are resolved using this specific framework. Changing this later after a lot of classes may use it is a headache.

To fix this, we need to isolate Swinject from our code more. One way of doing so is by extracting the dependencies that a component might need out into its own type:

```swift
class ListViewController {
  struct Dependencies {
    var apiService: APIService
    var detailCreator: () -> DetailViewController
  }

  let apiService: APIService
  let detailCreator: () -> DetailViewController

  init(dependencies: Dependencies) {
    self.apiService = dependencies.apiService
    self.detailCreator = dependencies.detailCreator
  }

  func showNext() {
    present(detailCreator())
  }
}
```

Now with our controller's `Dependency` type set, we can extend it in a secondary place to keep Swinject isolated from everything else:

```swift
extension ListViewController.Dependencies {
  init(resolver: Resolver) {
    apiService = resolver.resolve(APIService.self)!
    detailCreator = { resolver.resolve(DetailViewController.self)! }
  }
}
```

you can extrapolate this pattern to our other types too:

```swift
extension ResponseDeserializer.Dependencies {
  init(resolver: Resolver) {
    queue = resolver.resolve(DispatchQueue.self)
  }
}

extension APIService.Dependencies {
  init(resolver: Resolver) {
    urlSession = r.resolve(URLSession.self)!
    deserializer = r.resolve(ResponseDeserializer.self)!,
    environment = r.resolve(Environment.self)!
  }
}

extension DetailViewController.Dependencies {
  init(resolver: Resolver) {
    apiService = resolver.resolve(APIService.self)!
  }
}
```

And with this in place, we can simplify our dependency registration and usage by quite a lot:

```swift
func registerDependencies() -> Container {
  let container = Container()
  container.register(DispatchQueue.self, factory: { _ in DispatchQueue.global() })
  container.register(ResponseDeserializer.self, factory: { r in
    return ResponseDeserializer(dependencies: .init(resolver: r))
  })
  container.register(Environment.self, factory: { _ in
    return Environment(baseURL: URL(string: "https://example.com")!)
  })

  container.register(URLSession.self, factory: { _ in URLSession.shared })
  container.register(APIService.self, factory: { r in
    return APIService(dependencies: .init(resolver: r))
  })

  container.register(ListViewController.self, factory: { r in
    return ListViewController(dependencies: .init(resolver: r))
  })

  container.register(DetailViewController.self, factory { r in
    return DetailViewController(dependencies: .init(resolver: r))
  })

  return container
}
```

We can even write a custom registration method that will make it even shorter, if we make sure that we are extending each `Dependencies` type with a protocol:

```swift
protocol AutoResolving {
  init(resolver: Resolver)
}

protocol HasDependencies {
  associatedtype Dependencies: AutoResolving
}

extension Container {
  func register<Service: HasDependencies>(_ service: Service.Type) {
    register(service, factory: { r in service.Dependencies(resolver: r) })
  }
}

func registerDependencies() -> Container {
  let container = Container()
  container.register(DispatchQueue.self, factory: { _ in
    return DispatchQueue.global()
  })
  container.register(Environment.self, factory: { _ in
    return Environment(baseURL: URL(string: "https://example.com")!)
  })
  container.register(URLSession.self, factory: { _ in
    return URLSession.shared
  })
  container.register(ResponseDeserializer.self)
  container.register(APIService.self)
  container.register(ListViewController.self)
  container.register(DetailViewController.self)

  return container
}
```
