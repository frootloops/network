# The 5-th network layer
The main idea: we'll have *bunch* of providers that gives access to all our endpoints. Each provider will have domainand endpoint specific functions (for example: `NewsProvider.fetchNews(scoped: .wallet) { result: Result<[News], NSError> in }`). All non-endpoint specific work goes to several callbacks that you can pass to `Provider`. For example you can pass `responseClosure` block and have ability to execute code on each response. You can use the same approach to provide auth token and save models to CoreData.

# The main parts of the 5th network layer

## The providers

__The providers are the core of the layer.__ It's a instance of a class that describes all domain-specific (like News, Transactions) endpoints. That instance has a few callback that give you single point of access to all requests.

It provides to developers hight level API for access to all endpoints. It contains a bunch of functions that represent each endpoint.

Example of a provider:

``` swift
class OfferProvider {
init(endpointClosure: @escaping EndpointClosure,
                requestClosure: @escaping RequestClosure,
                plugins: [PluginType] = []) {}

    func viewOffer(offerId: Int, completion: @escaping (Result<Offer, NSError>) -> Void) -> Request {
	    ... 
    }
}

class FacebookProvider {
    ...
    
    func importFacebook(contacts: [String], completion: @escaping (Result<[ImportedContact], NSError>) -> Void) -> Request {
        ....
    }
}

class TransactionProvider {
    ....
}
```

Example of usage:

``` swift
let request = NewsProvider.fetchNews(scoped: .wallet) { result: Result<[News], NSError> in
    switch result {
    case .success(news):
        print(news) // array of News
    case .failure(error):
        print(error) // NSError
    }
}
```

In that case we know that `/api/news` returns an array of news model. So the provider will parse the response and create a bunch of __News__ models (that we defined before) and pass it to the completion handler.

Those methods do not have direct access to server response. It prevents us from putting logic inside completion block and force  us to do so inside provider (DRY pattern). If your endpoint returns two different type of models (for example it's an array of news and one user model) you just need pass a tuple into completion block.

``` swift
let request = provider.fetchNews(scoped: .wallet) { result: Result<(User, [News]), NSError> in
    switch result {
    case .success(user, news):
        print(user, news) // array of News
    case .failure(error):
        print(error) // NSError
    }
}
```


## The route

The route describes all backend endpoints. The route is a __enum__ where each case includes method (get, post, put), path (for example: `/v1/projects`) and query (for example: `name=arsen&age=26`).

``` swift
enum Route {
  case activities(categories: [Category])
  case addImage(fileUrl: URL, toDraft: UpdateDraft)
  case addVideo(fileUrl: URL, toDraft: UpdateDraft)
  case backing(projectId: Int, backerId: Int)
  
  var requestProperties:
    (method: Method, path: String, query: [String:Any], file: (name: UploadParam, url: URL)?) {

    switch self {
    case let .activities(categories, count):
      var params: [String:Any] = ["categories": categories.map { $0.rawValue }]
      params["count"] = count
      return (.GET, "/v1/activities", params, nil)

    case let .addImage(file, draft):
      return (.POST, "/v1/projects/\(draft.update.projectId)/updates/draft/images", [:], (.image, file))

    case let .addVideo(file, draft):
      return (.POST, "/v1/projects/\(draft.update.projectId)/updates/draft/video", [:], (.video, file))

    case let .backing(projectId, backerId):
      return (.GET, "/v1/projects/\(projectId)/backers/\(backerId)", [:], nil)
  }

}
```


## The models

A model is a struct that we use to represent any response from backend. Any response can be represented as one model or as an array of models.

Each model knows how to instantiate itself from JSON and how to represent itself to be saved into CoreData.

``` swift
struct News {
    let identifier: String
    let title: String
    ...

    enum Scope: String {
        case wallet
    }
}

extension News: ImmutableMappable {

    init(map: Map) throws {
        identifier  = try map.value("id")
        title       = try map.value("title")
        ...
    }

}

extension News: Storable {

    var table: String {
        return SwiftDatabaseManager.kEntityNews
    }

    var parameters: [String: Any] {
        return ["title": title]
    }

}


```

___

## Other topics

### Plug-ins
It's a very powerful mechanism that gives you ability to observe  all requests and responses. The most obvious case it's logging http requests.

```swift

class NetworkLoggerPlugin: PluginType {
    public func willSend(_ request: RequestType, target: TargetType) {
        print(request)
    }

    public func didReceive(_ result: Result<Response, NSError>, target: TargetType) {
        print(result)
    }
}

let provider = Provider(plugins: [NetworkLoggerPlugin()])
```

### Logout and 2FA cases
The provider is creating with a block that will be called when each response has arrived. It's a place where all logout/2FA logic live.

``` swift
let responseClosure = { (endpoint: Route, done: URLRequest, URLRespone -> Void) in
    let request = endpoint.urlRequest
    // now we can check status code of response and if it's a case we can logout the user
    
    // or if it's time to make 2FA we can hold `done` callback and when we know for sure that user authorised the request we can pass a response back to `done` callback
    
    2FA.signRequest(request, completion: { request, respone in
      done(request, respone)
    })
}
let provider = Provider(responseClosure: responseClosure)
```


### Objective-C compatibility
It's possible to make the provider compatible with Objective-C by providing alternative interface for APIs. For example:

``` swift
// swift API. It's not compatible with Objective-C
func fetchNews(scoped: News.Scope, completion: @escaping (Result<[News], NSError>) -> Void) -> Request

// But it can be represented as following:
func fetchNews(scoped: String, success: @escaping ([News]) -> Void, failure: @escaping (NSError) -> Void) -> Request

// Now it's compatible with Objective-C
```

### CoreData

Because each model implemented `Storable` protocol we can easily story all responses in CoreData. It can be done by something like this:

``` swift
let storeClosure = { (entities: [Storable]) in
    SwiftDatabaseManager.save(entities)
}
let provider = Provider(storeClosure: storeClosure)
```

---
The network layer will be distributed as a framework. It can be used within deferent targets such as iOS Application, watchOS application and iOS extensions.


