# Cropwise Operations-powerbi
PowerBI examples to work with Cropwise Operations HTTP API

> Using examples of PowerBI scripts requires installed PowerBI Desktop application.
> You can download this application from official [Microsoft page](https://powerbi.microsoft.com/en-us/desktop/).

# General workflow

## Step 1: Open PowerBI Desktop app and find **Edit Queries** navigation button as on the screenshot below:

![Edit Queries](images/edit_queries.png)

## Step 2: Find *New Source* navigation button and Select *Blank Query*.

![Add New Query](images/add_new_query.png)

## Step 3: Right click on created Query1 and select *Advanced Editor* option.

![Open Advanced Editor](images/press_advanced_editor.png)

## Step 4: Fill in the editor with your code and press *Done* button to complete.

![Put your code](images/complete_query.png)

## Step 5: Add Cropwise Operations API Token (X-User-Api-Token) to PowerBI credentials

PowerBI will request you to enter credentials to authrozie queries to Cropwise Operations API.
1. Press on **Edit credentials** button.
1. Press on **Web API** tab.
1. Insert Cropwise Operations API Token (X-User-Api-Token) into **Key** field.
1. You can create you API Token in  **https://operations.cropwise.com/user/linked_devices**.
1. Press **Connect** button to proceed.

PowerBI will save credentials. This step should be done only once.

![Add Cropwise Operations API Token to PowerBI Credentials](images/add_api_credentials.png)

## Step 6: After running script Select the column split icon in the ResourceName header. Select columns what you need and select **OK** button.

![Extract Table from Data rows](images/select_data_columns.png)

# Scripts examples

## Get full list of Fields




https://github.com/cropio/cropio-powerbi/assets/71318442/3cf079cf-7d51-4e9a-9fcc-b86cfda8eb9c




> NOTE: Without adding **X-User-Api-Token** to PowerBI Credentials, Query won't work.

> NOTE: You can choose any valid resource instead of `fields`. Example: `ResourceName = "agro_operations"`.
> List of available resources you can find at [Cropwise Operations API reference description](https://cropwiseoperations.docs.apiary.io/).
> NOTE: This example would iterate over pages in Cropwise Operations API to
> get full list of records (not only first 100 or 1000).

PowerBI example for list of Fields:
```PowerQuery
let
  // Cropwise Operations API reference: https://cropwiseoperations.docs.apiary.io/

  // Resource name from Cropwise Operations API reference.
  // Change to resource name you want to extract data from.
  ResourceName = "fields",

  // Additional query parameters for filtering/sorting.
  // Available filtering and soring options should be taken from Cropwise Operations API reference.
  // If not needed, leave empty record [].
  AdditionalQueryParameters = [
    // Add parameters here.
    // Example:
    // updated_at_gt_eq = "2023-01-01 00:00"
  ],

  // Base Cropwise Operations API URL
  BaseUrl = "https://operations.cropwise.com/api/",

  // Cropwise Operations API version. By default, "v3". Sometimes may be "v3a" or "v3b".
  // Should be taken from Cropwise Operations API reference.
  ApiVersion = "v3",

  // Relative resource path for API request.
  ResourceRelativePath = ApiVersion & "/" & ResourceName,

  GetJson = (QueryParams) =>
    let Options = [
      ApiKeyName = "user_api_token",
      RelativePath = ResourceRelativePath,
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
  Response = List.Generate(() => Json, each (GetObtainedRecordsCount(_)) > 0, each GetPage(GetLastRecordId(_))),
  Data = List.Transform(Response, each _[data]),
  DataToTable = Table.FromList(Data, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
  DataToColumn = Table.ExpandListColumn(DataToTable, "Column1"),
  ResultTable = Table.RenameColumns(DataToColumn,{{"Column1", ResourceName}})
in
  ResultTable
```

## Update Field record

> NOTE: Without adding **X-User-Api-Token** to PowerBI Credentials, Query won't work.

> NOTE: You can choose any valid resource instead of `fields`. Example: `ResourceName = "agro_operations"`.
>
> List of available resources you can find at [Cropwise Operations API reference description](https://cropwiseoperations.docs.apiary.io/).

> NOTE: As result of UPDATE request you will receive updated record.

PowerBI example to UPDATE Field name:
```PowerQuery
let
  // Cropwise Operations API reference: https://cropwiseoperations.docs.apiary.io/

  // Resource name from Cropwise Operations API reference.
  ResourceName = "fields",

  // ID of record we want to update in Cropwise Operations.
  // Example: ResourceID = "1",
  ResourceID = "VALID_ID_OF_FIELD",

  // Base Cropwise Operations API URL
  BaseUrl = "https://Cropwise Operations.com/api/",

  // Cropwise Operations API version. By default, "v3". Sometimes may be "v3a" or "v3b".
  // Should be taken from Cropwise Operations API reference.
  ApiVersion = "v3",

  // Data we want to update.
  // In this case we want set Field name to "MyFieldName123".
  ResourceContentToUpdate = "{ ""data"": { ""name"": ""MyFieldName123""} }",

  // UpdateResource function.
  UpdateResourceItem = (ResourceName, ResourceID, ContentToUpdate) =>
    // Relative resource path to send UPDATE request.
    let ResourceRelativePath = ApiVersion & "/" & ResourceName & "/" & Text.From(ResourceID),
    // Request Options.
    RequestOptions = [
      ApiKeyName = "user_api_token",
      Content = Text.ToBinary(ContentToUpdate),
      Headers = [
        #"Accept" = "application/json",
        #"Content-Type" = "application/json",
        #"X-HTTP-Method-Override" = "PATCH"
      ],
      RelativePath = ResourceRelativePath
    ],
    RawData = Web.Contents(BaseUrl, RequestOptions),
    Json = Json.Document(RawData)
    in Json,

  // Send UPDATE request and receive response.
  Source = UpdateResourceItem(ResourceName, ResourceID, ResourceContentToUpdate)[data]
in
  Source
```

## Update Machine Tasks records by table or JSON values list

> NOTE: Without adding **X-User-Api-Token** to PowerBI Credentials, Query won't work.

> NOTE: You can choose any valid resource instead of `machine_tasks`. Example: `ResourceName = "agro_operations"`.
>
> List of available resources you can find at [Cropwise Operations API reference description](https://cropwiseoperations.docs.apiary.io/).

> NOTE: In this example we updating `end_time` attribute. But you could update any other valid attributes.
>
> List of available attributes for each resource you can find at [Cropwise Operations API reference description](https://cropwiseoperations.docs.apiary.io/).

> NOTE: IN this example we use `MACHINE_TASKS` PowerBI table with `id` and `end_time` columns as input.
> Yo could use any other PowerBI table.

`MACHINE_TASKS` example table:
![Table Example](images/table_name_example.png)

PowerBI example to update Cropwise Operations Machine Tasks `end_time` values:
```PowerQuery
let
  // Cropwise Operations API reference: https://cropwiseoperations.docs.apiary.io/

  // Resource name from Cropwise Operations API reference.
  ResourceName = "machine_tasks",

  // Base Cropwise Operations API URL
  BaseUrl = "https://operations.cropwise.com/api/",

  // Cropwise Operations API version. By default, "v3". Sometimes may be "v3a" or "v3b".
  // Should be taken from Cropwise Operations API reference.
  ApiVersion = "v3",

  // UpdateResource function.
  UpdateResourceItem = (ResourceName, ResourceID, ContentToUpdate) =>
    // Relative resource path to send UPDATE request.
    let ResourceRelativePath = ApiVersion & "/" & ResourceName & "/" & Text.From(ResourceID),
    // Request Options.
    RequestOptions = [
      ApiKeyName = "user_api_token",
      Content = Text.ToBinary(ContentToUpdate),
      Headers = [
        #"Accept" = "application/json",
        #"Content-Type" = "application/json",
        #"X-HTTP-Method-Override" = "PATCH"
      ],
      RelativePath = ResourceRelativePath
    ],
    RawData = Web.Contents(BaseUrl, RequestOptions),
    Json = Json.Document(RawData)
    in Json,

  // Build JSON from end_time value
  ResourceContentToUpdateData = (data) =>
    let Res = "{ ""data"": { ""end_time"": """ & data & """} }"
    in Res,

  JsonDocument = Json.Document(Json.FromValue(MACHINE_TASKS)),
  RowCount = Table.RowCount(MACHINE_TASKS)
in
  List.Generate(
    () => 0,
    each _ < RowCount,
    each _ + 1,
    each UpdateResourceItem(
      ResourceName,
      JsonDocument{_}[MACHINE_TASKS.id],
      ResourceContentToUpdateData(JsonDocument{_}[MACHINE_TASKS.end_time])
    )[data]
  )
```

For multiple params update (`start_time` and `end_time`):

```PowerQuery
  ResourceContentToUpdateData = (start_time, end_time) =>
    let Res = "{
      ""data"": {
        ""start_time"": """ & start_time & """,
        ""end_time"": """ & end_time & """
      }
    }"
    in Res,

  JsonDocument = Json.Document(Json.FromValue(MACHINE_TASKS)),
  RowCount = Table.RowCount(MACHINE_TASKS)
in
  List.Generate(
    () => 0,
    each _ < count,
    each _ + 1,
    each UpdateResourceItem(
      ResourceName,
      JsonDocument{_}[MACHINE_TASKS.id],
      ResourceContentToUpdateData(
        JsonDocument{_}[MACHINE_TASKS.start_time],
        JsonDocument{_}[MACHINE_TASKS.end_time]
      )
    )[data]
  )
```

## Hints

### Create Table from JSON
```PowerQuery
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
  TableFromList = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
  TableExpandedRecords = Table.ExpandRecordColumn(TableFromList, "Column1", {"id", "end_time"}, {"Column1.id", "Column1.end_time"})
```

### Create JSON from Table
```PowerQuery
JsonDocument = Json.Document(Json.FromValue(TABLE_NAME))
```

## Work with PowerQuery in Excel

### Example for list of Fields:

```PowerQuery
let
  // Cropwise Operations API reference: https://cropwiseoperations.docs.apiary.io/

  // Resource name from Cropwise Operations API reference.
  // Change to resource name you want to extract data from.
  ResourceName = "fields",

  // Additional query parameters for filtering/sorting.
  // Available filtering and sorting options should be taken from Cropwise Operations API reference.
  // If not needed, leave empty record [].
  AdditionalQueryParameters = [
    // Add parameters here.
    // Example:
    // updated_at_gt_eq = "2021-01-01 00:00"
    //Paste you api token here
    user_api_token = "You api token"
  ],

  // Base Cropwise Operations API URL
  BaseUrl = "https://operations.cropwise.com/api/",

  // Cropwise Operations API version. By default, "v3". Sometimes may be "v3a" or "v3b".
  // Should be taken from Cropwise Operations API reference.
  ApiVersion = "v3",

  // Relative resource path for API request.
  ResourceRelativePath = ApiVersion & "/" & ResourceName,

  GetJson = (QueryParams) =>
    let Options = [
      ApiKeyName = "user_api_token",
      RelativePath = ResourceRelativePath,
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
  Response = List.Generate(() => Json, each (GetObtainedRecordsCount(_)) > 0, each GetPage(GetLastRecordId(_))),
  Data = List.Transform(Response, each _[data]),
  DataToTable = Table.FromList(Data, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
  DataToColumn = Table.ExpandListColumn(DataToTable, "Column1"),
  ResultTable = Table.RenameColumns(DataToColumn,{{"Column1", ResourceName}})
in
  ResultTable
```
### Create Chemical
PowerBI example to CREATE Chemical:
```PowerQuery
let
    url = "https://operations.cropwise.com/api/v3/chemicals",
    headers = [
        #"X-User-Api-Token" = "You api key",
        #"Content-Type" = "application/json"
    ],
    //Data we want to create
    data = "{""data"": {""name"": ""Roundup Super"", ""chemical_type"": ""herbicide"", ""units_of_measurement"": ""liter"", ""toxicity_class"": 1, ""action_term"": 5, ""action_term_units"": ""day"", ""active_substance"": ""100%"", ""drug_form"": ""liquid"", ""influence_method"": ""intestinal"", ""bees_isolating_recommended_term"": 10, ""bees_isolating_recommended_term_units"": ""day"", ""crop_ids"": [2920, 1158], ""sale_term"": ""2023-05-10"", ""term_of_use"": ""2023-02-11""}}",

    options = [
        Headers = headers,
        Content = Text.ToBinary(data)
    ],
    response = Json.Document(Web.Contents(url, options)),
  Navigation = response[data]
in
    Navigation
```

### Update Chemical record

> List of available resources you can find at [Cropwise Operations API reference description](https://cropwiseoperations.docs.apiary.io/).

> NOTE: As result of UPDATE request you will receive updated record.

PowerBI example to UPDATE Chemical name:
```PowerQuery

let
  // Cropwise Operations API reference: https://cropwiseoperations.docs.apiary.io/
    // Replace "123" with the unique identifier of the record you want to edit
    url = "https://operations.cropwise.com/api/v3/chemicals/123",
    headers = [
        #"X-User-Api-Token" = "You api token",
        #"Content-Type" = "application/json",
        #"X-HTTP-Method-Override" = "PATCH"
    ],
    // Data we want to update.
    // In this case we want set Chemical name to "Updated Name".
    data = "{""data"": {""name"": ""Updated Name""}}",
    options = [
        Headers = headers,
        Content = Text.ToBinary(data)
    ],
    response = Json.Document(Web.Contents(url, options)),
  Navigation = response[data]
in
    Navigation
```
