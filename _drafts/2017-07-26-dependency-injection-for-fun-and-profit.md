---
tags: swift dependency-injection
category: programming
---

If you've worked on a relatively large and established codebase for awhile, you may have encountered a large collection of singletons in the code. And if that code wasn't written in a testable way (via Test-Driven-Development or otherwise), then your singletons probably manage a whole bunch of state which probably makes any tests you try to write difficult to manage.

There's a great article by Miško Hevery called [Singletons Are Pathological Liars](http://misko.hevery.com/2008/08/17/singletons-are-pathological-liars/) which illustrates this problem beautifully. You should definitely give it a read, but the gist of the article is that singletons hide the complexities of your APIs so that it's very hard to understand what your code may be doing without a lot of trial and error. There's little that your tools can do to help you set up the state of your singletons properly, and it's incredibly easy to introduce problems as you refactor your code.

To fix this, you should employ a technique called Dependency Injection (or DI for short). All it means is that your types should never reference anything they may depend on unless those dependencies can be "injected" into instances of your type, such as via a constructor or a property. Lets look at a simple example where 

```swift
class APIService {
    let session: URLSession
    init(session: URLSession) {
        self.session = session
    }

    func request(path: String, parameters: [String: String], completion: ([String: Any]?) -> Void) {
        session.dataTask(with: request(for: path, with: parameters)) { (data, response, error) in
            guard let data = data else {
                completion(nil)
                return
            }

            let jsonDict = JSONSerialization.jsonObject(with: data, options: []) as! [String: Any]

            completion(jsonDict)
        }.resume()
    }
}

class Authenticator {
    let service: APIService
    init(service: APIService) {
        self.service = APIService
    }

    func login(username: String, password: String, completion: (Bool) -> Void) {
        let params = ["username": username, "password": password]
        service.request(path: "/login", parameters: params) { (json) in
            print(json)
        }
    }
}
```

It's a little bit more code, but now we can clearly see just by looking at the init methods and properties what this class needs to do its job. Furthermore, since we removed the `shared` static properties, any type that wants to use the Authenticator must do the same thing, which helps keep those types honest about their dependencies.

Now, how do you set this all up?
