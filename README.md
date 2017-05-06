# The main parts of the 5th network layer

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

A model is a struct that we use to represent any response from backend. Any response can be represent as one model or as an array of models.

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


## The provider

It provides to developers hight level API for access to all endpoints. It contains a bunch of functions that represent each endpoint.

``` swift
let request = Provider.fetchNews(scoped: .wallet) { result in
    switch result {
    case .success(news):
        print(news) // array of News
    case .failure(error):
        print(error) // NSError
    }
}
```

In that case we know that `/api/news` returns an array of news model. So the provider will parse the response and create a bunch of __News__ models (that we defined before) and pass it to the completion handler.

___

The network layer will be distributed as a framework. It can be used within deferent targets such as iOS Application, watchOS application and iOS extensions.


