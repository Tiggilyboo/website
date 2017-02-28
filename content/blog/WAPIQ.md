+++
date = "2017-02-25T22:48:15+13:00"
title = "WAPIQ: Web API Query Language"
description = "Introduction to a library I've written recently in golang. Explains what the project's all about, why I wrote it, and how to use it!"
draft = false
tags = ["Go", "API", "WAPIQ", "Parser"]
topics = ["Blog"]
comments=true
extralang=true
+++

## [WAPIQ](https://github.com/Tiggilyboo/wapiq)
*Web API Query Language*

### Introduction
After working on several project involving frequent web API response parsing and request assembling in many different front and back ends, you could say I've had enough of it... So I wrote a library in Golang which attempts to shorten the amount of time to map arbitrary JSON responses and setting up API request behaviours.

#### What does it look like?
I went for something of a hybrid between a query language like SQL, and that of a configuration or declarative format like JSON. This allows scoping dynamic naming without taking up too much space. We want something succinct enough, reusable, and easily maintainable if the API's response changes in the future).

### Setup
WAPIQ consists of several types of commands:

* `API` configures API endpoints, configured once before running queries off of.
* `GET` or `POST` configures HTTP requests from an API, specify query parameters, head, body, relative path and all that good stuff.
* `MAP` configures how to output the data from a `GET` or `POST` request.
* `QUERY` can be run separately after the above commands have been loaded. Queries the WAPI, returns the specified `MAP` response according to some `WHERE` conditions.

### Usage By Example

Below goes through a quick Google Places API example, more examples can be found in the [GitHub repository](http://github.com/Tiggilyboo/wapiq).


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

Right, so this is where it gets interesting: This creates a new response called `Place` with our previously created `GooglePlaces` API. Within this scope, we reference our declared `search` GET request. `Place` returns 5 fields: id, name, types, location, and address. If we were to map another response that still fills `search` we can add another scoped action within our MAP block. But for now, we map these fields with JSON paths.

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

### Querying
On to the good stuff, now that we've set up a simple WAPIQ configuration, let's query and return some results!

```wapiq
/search FOR Place WHERE
  name `cruise`
  location `-33.8670,151.1957`
  radius `500`
  types `food`
;
```

WAPIQ resembles SQL a lot here, we precede our HTTP action with a `/`, and we want to return a response `FOR` our mapped `Place`, under the conditions outlined after the `WHERE` keyword. In this case, we're looking for a cruise near the supplied lat and lon location with 500 as our radius... Oh and something that offers food.

`WHERE` is optional, if not arguments are given, the request sends with default arguments given in the associated API. In this case a query parameter named `key`. Alternatively, we could override the `key` argument with a different key in the query if we wanted to.

To sum up, that's the initial version of the library so far, you can access WAPIQ through a CLI, load your configurations through `*.wapiq` files, or execute and query through Golang using the WAPIQ library.

#### CLI
```sh
./wapiq -f some/configuration.wapiq -q="/search FOR Something WHERE conditions `plausible`;"
```

#### Go Wrapper
```go
import "github.com/Tiggilyboo/wapiq"
```

That's all for now, thanks for reading!
