# cropio-powerbi
PowerBI examples to work with Cropio HTTP API

> Using examples of PowerBI scripts requires installed PowerBI Desktop application.
> You can download this application from official [Microsoft page](https://powerbi.microsoft.com/en-us/desktop/).

## General workflow

#### Step 1: Open PowerBI Desktop app and find **Edit Queries** navigation button as on the screenshot below:

![Screenshot](/images/edit_queries.png)

#### Step 2: Find *New Source* navigation button and Select *Blank Query*.

![Screenshot](/images/add_new_query.png)

#### Step 3: Right click on created Query1 and select *Advanced Editor* option.

![Screenshot](/images/press_advanced_editor.png)

#### Step 4: Fill in the editor with your code and press *Done* button to complete request.

![Screenshot](/images/complete_query.png)

#### Step 5: After running script Select the column split icon in the ResourceName header. Select columns what you need and select **OK** button.

![Screenshot](/images/select_data_columns.png)


## Scripts examples

#### Obtaining API token with login and password

User must make Login action and obtain **X-User-Api-Token** token that required for all next requests. **X-User-Api-Token**  is a string, that should be added for all requests to Cropio API (https://cropioapiv3.docs.apiary.io/#reference/authentication/login-request).

PowerBI example:

```
let
  // Resource name from Cropio API reference.
  ResourceName    = "sign_in",

  // Base Cropio API URL
  BaseUrl         = "https://cropio.com/api/",

  // Cropio API version.
  ApiVersion      = "v3",
  
  // Relative resource path for API request.
  RelativeResourcePath = ApiVersion & "/" & ResourceName,

  // Important! Replace USER-EMAIL and USER-PASSWORD with your own credentials.
  RequestBody = "{ ""user_login"": { ""email"": ""USER-EMAIL"",  ""password"": ""USER-PASSWORD""} }",

  // Build Request.
  RequestOptions = [
    Headers = [
      #"Content-Type" = "application/json"
    ], 
    RelativePath = RelativeResourcePath,
    Content = Text.ToBinary(RequestBody)
  ],

  // Get data from Cropio API server.
  Response = Web.Contents(BaseUrl, RequestOptions),

  // Parse response from JSON.
  Source = Json.Document(Response)
in
  Source
```

#### Get list of resources

> Before making request you need to pass login action and receive **X-User-Api-Token**. 


You can choose any resource name instead of "field_shapes": `ResourceName = "field_shapes"`.
List of available resources you can find at [Cropio API reference description](https://cropioapiv3.docs.apiary.io/#reference).

PowerBI example (List of fields): 
```
let
  // Cropio API reference: https://cropioapiv3.docs.apiary.io/
  
  // Resource name from Cropio API reference.
  // Change to resource name you want to extract data from.
  ResourceName    = "field_shapes",
   
  // Important! Replace example value with your API token.
  Token           = "REPLACE WITH YOUR OWN X-User-Api-Token",
  
  // Additional query parameters for filtering/sorting.
  // Available filtering and soring options should be taken from Cropio API reference.
  // If not needed, leave empty record [].
  AdditionalQueryParameters = [
    // Add parameters here.
    // Example: 
    // updated_at_gt_eq="2021-01-01 00:00"
  ],
  
  // Base Cropio API URL
  BaseUrl         = "https://cropio.com/api/",
  
  // Cropio API version. By default, "v3". Sometimes may be "v3a" or "v3b".
  // Should be taken from Cropio API reference.
  ApiVersion      = "v3",
  
  // Relative resource path for API request.
  RelativeResourcePath = ApiVersion & "/" & ResourceName,
 
  GetJson = (QueryParams) =>
    let Options = [
      Headers=[
        #"X-User-Api-Token" = Token
      ],
      RelativePath = RelativeResourcePath,
      Query = QueryParams
    ],
    RawData = Web.Contents(BaseUrl, Options),
    Json = Json.Document(RawData)
    in Json,
    
  GetPage = (FromId) =>
    let QueryParams = AdditionalQueryParameters & [from_id=Text.From(FromId)],
    Json = GetJson(QueryParams)
    in Json,
 
  GetObtainedRecordsCount = (Json) =>
    let Count = Json[meta][response][obtained_records]
    in Count,

  GetRecordsLimit = (Json) =>
    let Limit = Json[meta][response][limit]
    in Limit,

  GetLastRecordId = (Json) =>
    let Id = Json[meta][response][last_record_id] + 1
    in Id,

  Json = GetPage(0),
  Response = List.Generate(() => Json, each (GetObtainedRecordsCount(_)) > 1, each GetPage(GetLastRecordId(_))),
  Data = List.Transform(Response, each _[data]),
  DataToTable = Table.FromList(Data, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
  DataToColumn = Table.ExpandListColumn(DataToTable, "Column1"),
  ResultTable = Table.RenameColumns(DataToColumn,{{"Column1", ResourceName}})
in
  ResultTable
```

#### Update resource item

> Before making request you need to pass login action and receive **X-User-Api-Token**. 


You can choose any resource name instead of "agro_operations": `ResourceName = "agro_operations"`.
Select agro operation what you want to update and write id instead of `RESOURCE_ITEM_ID`.
List of available resources you can find at [Cropio API reference description](https://cropioapiv3.docs.apiary.io/#reference).
As result you will receive updated resource item.

PowerBI example (Update Agro Operation status column): 
```
let
    ResourceName    = "agro_operations",
    Item_ID         = "RESOURCE_ITEM_ID",
    BaseUrl         = "https://cropio.com/api/v3/" & Text.From(ResourceName) & "/" & Text.From(Item_ID),
    Token           = "X-User-Api-Token",
    Method          = WebMethod.Put,
    ResourceContentToUpdate = "{ ""data"": { ""status"": ""done""} }",

    UpdateResourceItem = (Url, ContentToUpdate) =>
        let Options = [Headers=[#"X-User-Api-Token"=Token, accept="application/json", #"X-HTTP-Method-Override"="PATCH", #"Content-Type"="application/json"], Content=Text.ToBinary(ContentToUpdate)],
            RawData = Web.Contents(Url, Options),
            Json    = Json.Document(RawData)
        in  Json,
    Source = UpdateResourceItem(BaseUrl, ResourceContentToUpdate)[data]
in
    Source

```


#### Update resource items by table or json values list

> Before making request you need to pass login action and receive **X-User-Api-Token**. 

You can change `TABLE_NAME` for any of your's tables

`TABLE_NAME` example table with `id` and `end_time` columns:
![Screenshot](/images/table_name_example.png)

You can choose any resource name instead of "machine_tasks": `ResourceName = "machine_tasks"`.
You can choose any param name instead of `end_time`: 
`Res = "{ ""data"": { ""end_time"": """ & data & """} }"`.

List of available resources you can find at [Cropio API reference description](https://cropioapiv3.docs.apiary.io/#reference).
As result you will receive updated resource item.

PowerBI example (Update Machine Task end_time column): 
```
let
    ResourceName    = "machine_tasks",
    BaseUrl         = "https://cropio.com/api/v3/" & Text.From(ResourceName) & "/",
    Token           = "X-User-Api-Token",
    Method          = WebMethod.Put,

    UpdateResourceItem = (Url, ContentToUpdate) =>
        let Options = [Headers=[#"X-User-Api-Token"=Token, accept="application/json", #"X-HTTP-Method-Override"="PATCH", #"Content-Type"="application/json"], Content=Text.ToBinary(ContentToUpdate)],
            RawData = Web.Contents(Url, Options),
            Json    = Json.Document(RawData)
        in  Json,

    ResourceContentToUpdateData  = (data) =>
        let Res = "{ ""data"": { ""end_time"": """ & data & """} }"
        in Res,

    json1 = Json.Document(Json.FromValue(TABLE_NAME)),
    count = Table.RowCount(TABLE_NAME)
in
    List.Generate(() => 0, each _ < count, each _ + 1, each UpdateResourceItem((BaseUrl & json1{_}[TABLE_NAME.id]), ResourceContentToUpdateData(json1{_}[TABLE_NAME.end_time]))[data])

```
For multiple params update:

```
 ResourceContentToUpdateData  = (start_time, end_time) =>
    let Res = "{
         ""data"": {
            ""start_time"": """ & start_time & """,
            ""end_time"": """ & end_time & """
         } 
    }"
    in Res,

    List.Generate(() => 0, each _ < count, each _ + 1, each UpdateResourceItem((BaseUrl & json1{_}[TABLE_NAME.id]), ResourceContentToUpdateData(json1{_}[TABLE_NAME.start_time], json1{_}[TABLE_NAME.end_time]))[data])
```

#### Hints

###### Create Table from JSON
```
    Source = Json.Document("[
    {
        ""id"":""1"",
        ""end_time"":""2019-11-29T14:00:00.000+02:00"",
    },
    {
        ""id"":""2"",
        ""end_time"":""2016-05-28T19:00:00.000+03:00"",
    }
    ]"),
    #"Преобразовано в таблицу" = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    #"Развернутый элемент Column1" = Table.ExpandRecordColumn(#"Преобразовано в таблицу", "Column1", {"id", "end_time"}, {"Column1.id", "Column1.end_time"})
```

###### Create JSON from Table
```
json1 = Json.Document(Json.FromValue(TABLE_NAME))
```
