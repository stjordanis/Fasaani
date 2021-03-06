# Fasaani

<p align="center">
<img src="https://github.com/ssiltanen/Fasaani/raw/master/Logo.png" width="200px"/>
</p>

Fasaani is a simple F# wrapper on top of Azure Search .NET SDK. It does not try to cover all functionality of the wrapped SDK, but instead it covers some common use cases of querying Azure Search indexes and offers an easy way to use the raw SDK underneath while still using the nicer syntax of Fasaani.

## Installation

Download the assembly from Nuget [https://www.nuget.org/packages/Fasaani](https://www.nuget.org/packages/Fasaani) and open namespace Fasaani in your code.

```fsharp
open Fasaani
```

## Overview of supported settings

```fsharp
query {
    searchText "text used to search"    // Text searched from index
    searchFields [ "field" ]            // Which fields are matched with text search
    searchMode All                      // Whether any or all of the search terms must be matched. All or Any
    querySyntax Simple                  // Configure query to use simple syntax or Lucene syntax. Simple or Lucene
    facets [ "field"; "field2" ]        // Facets to return, see facets
    filter (where "field" Eq 1)         // Query filter, see filtering
    skip 10                             // Skip count of results, see paging
    top 25                              // How many results are returned, see paging
    order [ byField "field2" Asc ]      // Order of returned results, see paging
    includeTotalResultCount             // Return total matched results count, see paging
    requestOptions opt                  // Raw request options of underlying SDK, see Help! section
    //parameters par                    // Raw parameters of underlying SDK. Overwrites above other settings! See Help! section
} |> searchAsync<'T> indexClient
```

**All above settings are optional so use only the ones you need in your query.** More info of each can be found below. 

The parameters setting is commented out in the example to emphasize that it **should not** be used together with other settings as **they overwrite each other** if used in the same query. This is supported to allow easy way to fall back to raw SDK, but using it is not recommended unless needed. 

Extra mention also to Select parameter that sets the returned fields from result documents in the underlying library which is implicitly implemented in Fasaani from the output model properties.

## How to use

### Basic query

First create SearchIndexClient like you normally would. Notice that you should use the secondary key as it has less privileges.

```fsharp
use indexClient = new SearchIndexClient (searchName, indexName, SearchCredentials(searchQueryKey)) :> ISearchIndexClient
```

After creating the client, define a basic text search query with Fasaani `query` computational expression, and pipe it to `searchAsync` expression to execute your query:

```fsharp
query {
    searchText "text used to search"
} |> searchAsync<'T> indexClient

// Returns
// type SearchResult<'T> =
//     { Results: 'T seq
//       Facets: Map<string, string seq>
//       Count: int64 option
//       Raw: DocumentSearchResult<'T> }
```

Notice that you need to provide `searchAsync` expression with a type parameter to determine what type of results is returned.

In the response model, `Raw` contains the full as-is response the underlying SDK returned. This is returned to make sure that using this library does not prevent some use cases that are not pre-parsed for the user. The other three fields `Results`, `Facets`, and `Count` are pre-parsed from the underlying response for easier access in F#.

### Query with filtering

Fasaani offers simple way to create filters for your queries. The easiest way to create a filter is to use the `where` function:

```fsharp
let filter = where "MyField" Eq "Value"
let filter2 = where "NumericField" Gt 5
let filter3 = where "MyField" Eq null
// etc..
```

The supported comparison operators are `Eq`, `Ne`, `Gt`, `Lt`, `Ge`, and `Le`. 

If Fasaani does not support a type of filter you need, it is also possible to create your raw OData filter:

```fsharp
Filter.OData "MyField eq 'Value'"
```

To combine filters into multi-filter expression, Fasaani has two helper functions: `combineWithAnd`, and `combineWithOr`. For example:

```fsharp
[ where "MyField" Eq "Value"
  where "AnotherField" Eq "AnotherValue"
  where "NumericField" Gt 5 ]
|> combineWithAnd

[ where "MyField" Eq "Value"
  where "AnotherField" Eq "AnotherValue"
  where "NumericField" Gt 5 ]
|> combineWithOr
```

If you want more freedom over how these are combined you can also combine them by folding. Filters have two custom operators `(+)` = and, and `(*)` = or, which can be used to combine filters:

```fsharp

[ where "MyField" Eq "Value"
  where "AnotherField" Eq "AnotherValue"
  where "NumericField" Gt 5 ]
|> List.fold (+) Filter.Empty

// Or withot fold
(where "MyField" Eq "Value") + (where "AnotherField" Eq "AnotherValue")
```

It is also possible to create a filter for OData `search.in` function with helpers `whereIsIn` and `whereIsInDelimited`:

```fsharp
[ whereIsIn "MyField" [ "Value1"; "Value2" ]
  whereIsInDelimited "AnotherField" [ "Value with spaces and ,"; "Another value" ] '|' ] // Just an example character as pipe
|> combineWithAnd
```

To negate the filter you can use either `isNot` or operator `(!!)`

```fsharp
where "MyField" Eq "Value" |> isNot
// or
!! (where "MyField" Eq "Value")
```

After forming the filter, provide it to the `query` CE:

```fsharp
let filterExpr =
    [ where "MyField" Eq "Value"
      where "AnotherField" Eq "AnotherValue"
      where "NumericField" Gt 5 ]
    |> List.fold (+) Filter.Empty

query {
    filter filterExpr
} |> searchAsync<'T> indexClient

// Or with some text search
query {
    searchText "Some text to search with"
    filter filterExpr
} |> searchAsync<'T> indexClient
```

#### Geo functions

Fasaani supports filtering with OData geo functions geo.distance and geo.intersect with functions `whereDIstance` and `whereIntersects`. They can be used together with `where` function or alone as a filter

```fsharp
let distanceFilter = whereDistance "field" (Lat -122.131577M, Lon 47.678581M) Lt 1

let intersectFilter =
    whereIntersects 
        "field"
        [ Lat -122.031577M, Lon 47.578581M
          Lat -122.031577M, Lon 47.678581M
          Lat -122.131577M, Lon 47.678581M
          Lat -122.031577M, Lon 47.578581M ]
```

Notice in the whereIntersects example that Azure Search has rules for the coordinate sequence:
 
- At least 3 unique coordinate points
- Points in a polygon must be in counterclockwise order
- The polygon needs to be closed, meaning the first and last point sets must be the same.

For now Fasaani does not provide any automatisation for these, but they are planned for the future.

### Paging and order by

Azure search incorporates a default skip and top values. Currently skip = 0 and top 50. These values can be overwritten with Fasaani with `skip`, and `top` operators.

Order by fields are specified with a collection of either field based order, distance based order or search score based order.

By default Azure Search does not calculate the total count of found documents for the search, and it needs to be specified explicitly with `includeTotalResultCount` operator:

```fsharp
query {
    searchText "Search text"
    skip 0
    top 100
    order [ byField "field1" Asc; byDistance "field2" (Lat -122.131577M, Lon 47.678581M) Desc; bySearchScore Desc ]
    includeTotalResultCount
} |> searchAsync<'T> indexClient
```

### Facets

Specify collection of facets to return them in the response

```fsharp
query {
    searchText "Search text"
    facets [ "field1"; "field2" ] // Response Facets are empty unless these are provided
} |> searchAsync<'T> indexClient
```

In the return value, facets are returned as a `Map<string, string seq>` where keys are the field names specified in the query facets above (in this case field1 and field2). and payload contains unique values of that field.

### Search mode

To specify whether any or all of the search terms must match in order to count the document as a match. Possible values include: `Any`, `All`

```fsharp
query {
    searchText "Search text"
    searchMode All
} |> searchAsync<'T> indexClient
```

### Search query syntax aka query mode

To specify query mode i.e. the syntax used in the query, provide querySyntax value of possible values `Simple`, `Lucene`.

```fsharp
query {
    searchText "Search text"
    querySyntax Lucene
} |> searchAsync<'T> indexClient
```

### Select

Azure Search .NET SDK allows setting select fields aka fields that are returned of found documents. Using this parameter is not necessary in Fasaani, as Fasaani uses reflection on the type parameter provided and sets them as query select field values.

### Search fields

To limit which fields Azure Search uses for text search, provide search fields in a collection. When no search fields are specified, all document fields are searched.

```fsharp
query {
    searchText "Search text"
    searchFields [ "field1"; "field2" ]
} |> searchAsync<'T> indexClient
```

### Special constants

Fasaani maps F# float special values (`nan`, `infinity`, `infinity`) to OData counterpars (NaN, INF, -INF)

These can be used in filters for example:

```fsharp
let filter =
    [ where "field1" Lt infinity
      where "field2" Gt -infinity
      where "field3" Ne nan ]
```

### Query configuration

So far in each example we have used the `searchAsync` function. This function is the non-configurable version of search function. To customize how the search is executed, use `SearchWithConfigAsync`, which can be configured to logger function and/or cancellation token.

Logging with config record works by creating a record of `SearchConfig` with `Log` value. This field is a function that is called before query execution if it is provided. For now it is user's responsibility to provide the implementation for that function in a form that pleases them.

Cancellation token works in the same fashion as logging. If `CancellationToken` is provided with `SearchConfig` record, it is passed to the underlying SDK. Otherwise CancellationToken.None is passed.

For Example:

```fsharp
// Type specified here for documentation purposes
type SearchConfig =
    { Log: (string option -> SearchParameters -> SearchRequestOptions option -> unit) option
      CancellationToken: CancellationToken option }

// Example of how to specify the config record
let configWithLogging =
    { Log =
        fun str parameters options ->
            str |> Option.iter log.LogInformation // Logging provider not specified here
            parameters.Filter |> Option.ofObj |> Option.iter log.LogInformation
        |> Some
      CancellationToken = None }

query {
    searchText "Search text"
    filter (where "MyField" Eq "Bear")
} |> searchWithConfigAsync<'T> indexClient configWithLogging
```

## Help! Fasaani does not support my use case

To mitigate the small set of features in Fasaani, it is possible to overwrite search parameters, and search request options of the query with `parameters`, and `requestOptions` operators, while still using the Fasaani query syntax:

```fsharp
let customParameters = SearchParameters()
customParameters.Top <- Nullable 100
let customRequestOptions = SearchRequestOptions()

query {
    searchText "Search text"
    parameters customParameters
    requestOptions customRequestOptions
} |> searchAsync<'T> indexClient
```

In addition, the raw `DocumentSearchResult<'T>` object from underlying method is also returned as `Raw` in the response body for access to values not offered by Fasaani.