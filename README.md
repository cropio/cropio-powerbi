# cropio-powerbi
PowerBI examples to work with Cropio HTTP API

> Using examples of PowerBI scripts requires installed PowerBI Desktop application.
> You can download this application from official [Microsoft page](https://powerbi.microsoft.com/en-us/desktop/).
> Then open PowerBI Desktop app and find **Edit Queries** navigation button as on the screenshot bellow:

![Screenshot](/images/edit_queries.png)

### Obtaining API token with login and password

User must make Login action and obtain **X-User-Api-Token** token that required for all next requests. **X-User-Api-Token**  is a string, that should be added for all requests to Cropio API (https://docs.cropioapiv3.apiary.io/#reference/authentication/login-request).

PowerBI example:

```
let
    url = "https://cropio.com/api/v3/sign_in",
    body = "{ ""user_login"": { ""email"": ""USER-EMAIL"",  ""password"": ""USER-PASSWORD""} }",
    Parsed_JSON = Json.Document(body),
    BuildQueryString = Uri.BuildQueryString(Parsed_JSON),
    Source = Json.Document(Web.Contents(url,[Headers = [#"Content-Type"="application/json"], Content = Text.ToBinary(body) ] ))
in
    Source
```

### Get list of resources

> Before making request you need to pass login action and receive **X-User-Api-Token**. 


You can choose any resource name instead of "fields": `ResourceName = "fields"`.
List of available resources you can find at [Cropio API reference description](https://docs.cropioapiv3.apiary.io/#reference).

PowerBI example (List of fields): 
```
let
    ResourceName    = "fields",
    BaseUrl         = "https://cropio.com/api/v3/" & Text.From(ResourceName),
    Token           = "cxiVFSbK3dPj7WbFyzQi",
 
    GetJson = (Url) =>
        let Options = [Headers=[ #"X-User-Api-Token" = Token ]],
            RawData = Web.Contents(Url, Options),
            Json    = Json.Document(RawData)
        in  Json,
 
    GetObtainedRecordsCount = (Json) =>
        let Count = Json[#"meta"][response][obtained_records]
        in  Count,

    GetRecordsLimit = (Json) =>
        let Limit = Json[#"meta"][response][limit]
        in  Limit,    
 
    GetLastRecordId = (Json) =>
        let Id = Json[#"meta"][response][last_record_id]
        in  Id,

    GetPage = (FromId) =>
        let FromId  = "?from_id=" & Text.From(FromId),
            Url   = BaseUrl & FromId,
            Json  = GetJson(Url)
        in  Json,

    Json = GetPage(0),
    Response = List.Generate(() => Json, each (GetObtainedRecordsCount(_)) > 1, each GetPage(GetLastRecordId(_))),
    Data = List.Transform(Response, each _[data])
in
    Data
```
