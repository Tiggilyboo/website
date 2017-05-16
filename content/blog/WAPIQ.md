+++
date = "2017-02-25T22:48:15+13:00"
title = "WAPIQ: Web API Query Language"
description = "Introduction to a library I've written in golang. Explains what the project's all about, why I wrote it, and how to use it!"
draft = false
tags = ["Go", "API", "WAPIQ", "Parser"]
topics = ["Blog"]
comments=true
extralang=true
+++

### <a href="https://github.com/Tiggilyboo/wapiq"><span class="fa fa-github"></span> Open Sourced On Github</a>

### Introduction
After working on several project involving frequent web API interactions, including: response parsing and request assembling, you could say I've had enough of it... So I wrote a library/query language in Golang which attempts to shorten the amount of time to map arbitrary JSON responses and setting up web API request behaviours.

#### What does it look like?
I went for something of a hybrid between a query language like SQL, and that of a configuration or declarative format like JSON. This allows scoping dynamic naming without taking up too much space: we want something succinct enough, reusable, and easily maintainable if the API's response changes in the future.

### Setup
WAPIQ consists of several types of command keywords:

* `API` configures API endpoints, configured once before running queries off of.
* `GET` or `POST` configures HTTP requests from an API, specify query parameters, head, body, relative path and all that good stuff.
* `MAP` configures how to output the data from a `GET` or `POST` request.
* `/<action>` can be run separately after the above commands have been loaded. Queries the WAPI, returns the specified `MAP` response according to some `WHERE` conditions.

### Usage By Example

Below goes through a quick Google Places API example, more examples can be found in the [GitHub repository](https://github.com/Tiggilyboo/wapiq/tree/master/examples).

#### Configuration

```wapiq
# Create new API configuration to interact with Google Places
"GooglePlaces" API {
  path `https://maps.googleapis.com/maps/api/place/`
  args {
    "key" `YOUR_API_KEY`
  }
};
```
So, the above WAPIQ snippet declares a new API called `GooglePlaces`, with a base path, and a argument store for any later requests that use this API (more on this later).
Also, notice that `#` is our full-line comment character.

**Example Request Configuration**
```wapiq
# Create a new GET http request
"search" GET {
  path `nearbysearch/json`
  type `json`
  query [
    `key`
    `location`
    `radius`
    `types`
    `name`
  ]
};
```

Similar in syntax to the previous example; this snippet creates a new HTTP GET request called `search`. It has a path which gets appended to the API path, sets the expected return type, and the next bit declares possible `query` parameters that can be added to the URL for any future `search` request made.

Now that we have set up a very basic API and action for it (Also: Notice how each request configuration does not explicitly reference the API, so we could use it on multiple APIs if you wanted to).
So, we want to `MAP` our API requests to a format of our liking. To do this we use the `MAP` configuration.

**Example Map Configuration**
```wapiq
# Create a new MAP between Place and our search request
"Place" MAP "GooglePlaces" {
  "search" {
    "id"        `results.place_id`
    "name"      `results.name`
    "types"     `results.types`
    "location"  `results.geometry.location`
    "address"   `results.vicinity`
  }
};
```

Right, so this is where it gets interesting: This creates a new response called `Place` with our previously created `GooglePlaces` API. Within this scope, we reference our declared `search` GET request. `Place` returns 5 fields: id, name, types, location, and address. If we were to map another response that still fills `search` we can add another scoped action within our MAP block. But for now, we will just be mapping `search` fields with JSON paths.

**JSON Paths?**
Let's use the below as an example:
```wapiq
"location" `results.geometry.location`
```
Alright, so in our standard Google Places API response, we expect a JSON response that looks similar to this:
```json
{
  "results": [
    ...
    "geometry": {
      ...
      "location": {
        "lat": "some_float",
        "lon": "some_float",
      }
    },
    ...
  ]
}
```
Our JSON path string will find all objects satisfying any object in `results`, in each result, a `geometry` object, in each geometry, a `location`, and finally in each location a `lat` and `lon` value. - Quite powerful and easy to change in the future if the API response changes in the future.

**Handling JSON Array elements using @ Expressions**

Some APIs use arrays with an expected set of information (without naming the key of the value in the pair). In order to handle these responses, WAPIQ uses the `@` character when specifying the index in the array. Here is an example that is used when handling responses from the Bitfinex exchange API (See more in the giy repo at ```examples/Bitfinex.wapiq```):

```wapiq
...
"Trade" MAP "Bitfinex" {
  "trades" {
    "Id"              @0,0
    "Mts"             @0,1
    "Amount"          @0,2
    "Price"           @0,3
  }
  "funding" {
    "Id"              @0,0
    "Mts"             @0,1
    "Amount"          @0,2
    "Rate"            @0,3
    "Period"          @0,4
  }
};
```

In the above example, ```@0,0``` is used to read a json value from a response like this one, where <> are used as placeholders to identify each embedded array indexes (if it was `@0` we take the first object in the array, each concecutive `,` between values gives the next index as the child). Here is an example:

```json
  [
    [
      <ID>,
      <MTS>,
      <AMOUNT>,
      <PRICE>
    ],
    [
      ...
    ],
    ...
  ]
```

The first ```@0``` specifies the first embedded array object, ```[[<here>],...]```, and the second identifies the first element (at index 0) inside that array.

If we wanted to just read the first element in an array, we just use ```@0```, and ```@1``` is the next element, etc.

**File Includes**

So let's say you're implementing a particularily intricate API using WAPIQ, but you are struggling with the separation of concerns of each script. For example, you are implementing a public or common portion of the API, and want to separate the authentication endpoints. In order to handle this, you can use include statements using a preceding `^` character followed by the filename (excluding the `.wapiq` filename extension).

As an example, the Bitfinex example is split into Bitfinex.wapiq and Bitfinex_Auth.wapiq:
```wapiq
# Include common API from Bitfinex.wapiq
^Bitfinex

"wallets" POST {
  path "auth/wallets"
  head [
    `bfx-apikey`
    `bfx-apisecret`
    `bfx-nonce`
  ]
};
```

At the moment WAPIQ does not support inheritance of objects (potential in the future), but this at least allows file separation.

### Querying
On to the good stuff, now that we've set up a WAPIQ configuration, let's query and return some results!

```wapiq
/search FOR Place WHERE
  name `cruise`
  location `-33.8670,151.1957`
  radius `500`
  types `food`
;
```

WAPIQ resembles SQL a lot here, we precede our query with a `/`, and we want to return a response `FOR` our mapped `Place` , under the conditions outlined after the `WHERE` keyword. In this case, we're looking for a cruise near the supplied lat and lon location with 500km as our radius... Oh and something that offers food.

The `WHERE` clause is optional - if no arguments are given the request sends with default arguments given in the associated API args (if any are given). In this case a query parameter named `key` was declared in our `GooglePlaces` example. Alternatively, we could override the `key` argument with a different value if you so choose.

Behind the scenes, this fires off a request to:

> https://maps.googleapis.com/maps/api/place/nearbysearch/json?key=AIzaSyCZmDlZXIlhlkDbHzAfffvWGWQa1LliZvE&location=-33.8670%2C151.1957&name=cruise&radius=500&types=food

Which, depending on if you invoked from the Go wrapper or CLI will resolve to the following JSON or go struct:

```json
{  
   "0":[  
      {  
         "address":"32 The Promenade, King Street Wharf 5, Sydney",
         "id":"ChIJrTLr-GyuEmsRBfy61i59si0",
         "location":{  
            "lat":-33.867591,
            "lng":151.201196
         },
         "name":"Australian Cruise Group",
         "types":[  
            "travel_agency",
            "restaurant",
            "food",
            "point_of_interest",
            "establishment"
         ]
      },
      {  
        ...
      }
   ]
}
```

To sum up, that's the initial version of the library so far, you can access WAPIQ through a CLI, load your configurations through `*.wapiq` files, or execute and query through Golang using the WAPIQ library.

#### CLI
```sh
./wapiq -f some/configuration.wapiq -q="/search FOR Something WHERE conditions `plausible`;"
```

* ```-q```: Run a query directly within the CLI.
* ```-f```: Load and execute an external wapiq file.

#### Go Wrapper
```go
package main
import "github.com/Tiggilyboo/wapiq"

func main(){
  w := &wapiq.WAPIQ{}
  w = w.New()

  w.Load("../examples/GooglePlaces.wapiq", false)

  // r is a map[string]interface{} and can be casted directly into GoLang models
  r := w.Query("/Search FOR Places;")

  // Alternatively, output into JSON, which could be serialized into another interface/language if needed
  fmt.Println(r.JSON())
}
```

That's all for now, thanks for reading!
