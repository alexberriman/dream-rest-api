<h1 align="center" id="top">
  Dream Rest API ✨
  <br>
</h1>
<h4 align="center">An opinionated guide on how to design beautiful REST APIs</h4>

<div align="center">

[![twitter](https://img.shields.io/badge/Twitter-1DA1F2?logo=twitter&logoColor=white)](https://twitter.com/bezz) [![github](https://img.shields.io/badge/GitHub-100000?logo=github&logoColor=white)](https://github.com/alexberriman/) [![youtube](https://res.cloudinary.com/practicaldev/image/fetch/s--cumRvkw3--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://img.shields.io/badge/YouTube-FF0000%3Flogo%3Dyoutube%26logoColor%3Dwhite)](https://www.youtube.com/@alex.berriman) [![linkedin](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/alex-berriman/)

</div>

## Table of contents

- [Introduction](#introduction)
- [Basic conventions](#basic-conventions)
  - [HTTP verbs](#http-verbs)
  - [HTTP response codes](#http-response-codes)
  - [URL structure](#url-structure)
  - [Resources](#resources)
  - [Response body](#response-body)
- [Operations](#operations)
  - [Create](#create)
  - [Read](#read)
  - [Update](#update)
  - [Delete](#delete)
- [Errors](#errors)
- [Filtering](#filtering)
- [Sorting](#sorting)
- [Pagination](#pagination)
  - [Cursor based pagination](#cursor-based-pagination)
  - [Page based pagination](#page-based-pagination)
- [Expanding relations](#expanding-relations)
- [Partial responses](#partial-responses)
- [Caching](#caching)
  - [ETag](#etag)
  - [If-Modified-Since](#if-modified-since)
- [Versioning](#versioning)
- [Reserved query parameters](#reserved-query-parameters)

## Introduction

Over the years I've both written and interfaced with hundreds of APIs. Some have been great - a lot, however have not. One of the great things about REST is that it's flexible. While there are certain community conventions and best practices in place, ultimately it allows you to design APIs however you want. With great power though comes great responsibility 🕷. Unfortunately, this flexibility often gives rise to poorly designed, inconsistent APIs that are frustrating to interface with.

The purpose of this repository is to document what I consider to be my dream API. That is, an API that I, as a software engineer would love to consume.

Before reading on, I want to throw out a quick disclamer. I obviously don't think there's a single _best_ way to design APIs. My views have evolved over time and I expect them to continue to do so. I don't expect everyone to agree on everything. Feel free to pick and choose different parts of this guide if you find them useful. Better yet, if there's something you disagree with or areas of improvement you see, please raise an issue and let's discuss it.

<div align="right"><a href="#top">Back to top</a></div>

## Basic conventions

### HTTP verbs

Standard HTTP verbs should be used:

| HTTP verb | When to use                                          |
| --------- | ---------------------------------------------------- |
| `GET`     | Retrieve a list of resources, or a specific resource |
| `POST`    | Create a new resource                                |
| `PUT`     | Replace a resource                                   |
| `PATCH`   | Partially update a resource                          |
| `DELETE`  | Delete a resource                                    |

<div align="right"><a href="#top">Back to top</a></div>

### HTTP response codes

Standard HTTP response codes should be used. `2xx` responses should indicate a success, `4xx` a client error and `5xx` a service error:

| HTTP code                   | When to use                                                                                          |
| --------------------------- | ---------------------------------------------------------------------------------------------------- |
| `200 OK`                    | The request was a success and the service has data to return                                         |
| `201 Created`               | A new resource was successfully created                                                              |
| `204 No Content`            | The request was successful but the service has no data to return                                     |
| `400 Bad Request`           | The client request was invalid                                                                       |
| `401 Unauthorized`          | The client was not authorized to perform the request                                                 |
| `404 Not Found`             | The requested resource does not exist                                                                |
| `405 Method Not Allowed`    | The HTTP method is not supported on the requested endpoint                                           |
| `412 Precondition Failed`   | Access to the target resource has been denied                                                        |
| `500 Internal Server Error` | A generic server error that should only be returned when a more descriptive error cannot be inferred |

<div align="right"><a href="#top">Back to top</a></div>

### URL structure

When interfacing with a single instance of resource, use the singular spelling variation (e.g. `user`). When interfacing with the collection, use a plural (e.g. `users`).

| URL                                | Description                            | Example            |
| ---------------------------------- | -------------------------------------- | ------------------ |
| `GET /{resource:singular}/{id}`    | Retrieve a single resource by ID       | `GET /user/abc`    |
| `GET /{resource:plural}`           | Query and retrieve a list of resources | `GET /users`       |
| `POST /{resource:plural}`          | Create a new resource                  | `POST /users`      |
| `PUT /{resource:singular/{id}`     | Replace a resource by ID               | `PUT /user/abc`    |
| `PATCH /{resource:singular/{id}`   | Partially update a resource by ID      | `PATCH /user/abc`  |
| `DELETE /{resource:singular}/{id}` | Delete a resource by ID                | `DELETE /user/abc` |

Unless there is a strong case not to, you should strive to use non-enumerable unique IDs (such as a uuid). Auto incrementing IDs should be an exception, not the norm.

#### Relations

Where it makes sense, you can use nested URLs to return related data: `/{resource:singular}/{id}/{relation}`.

- When the relation is 1:1, use the singular spelling variation (`GET /user/{id}/profile`)
- When the relation is 1:n, use the plural (`GET /user/{id}/transactions`)

Make sure to set a limit in respect to how deep you can nest URLs. To keep things simple, I tend to limit to a single level of nesting:

- Good ✅: `/user/{id}/transactions`
- Bad ❌: `/user/{id}/transactions/{transactionId}/products/{productId}`

In situations where you need to query on related resources, encourage consumers to query the resource directly, e.g.:

```
GET /transactions?user={userId}&product={productId}
```

<div align="right"><a href="#top">Back to top</a></div>

### Resources

#### Naming

Resources and their properties should be named using `camelCase` (e.g. `createdAt` instead of `created_at`).

#### Required properties

- A resource should always have a read-only property that specifies when it was created (e.g. `createdAt`)
- A resource should always have a read-only property that specifies when it was last updated (e.g. `updatedAt`)

#### Date format

Use [ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html) to represent all dates and times (e.g. `2020-01-01T00:00:00.000Z`) For simplicity, store all datetimes with a zero offset (that is, store in UTC time). If you need to store timezone information, do it in a separate property. A user's timezone may change for example, but the datetime they completed an action will not.

<div align="right"><a href="#top">Back to top</a></div>

### Response body

Data envelopes are unnecessary and should not be used. For a single resource, return the object directly. For multiple resources, return an array of objects. If you need to return additional information, use custom HTTP headers.

#### Example with data envelope (bad ❌)

```json
{
  "data": {
    "id": "aaa",
    "name": "Albert Einstein"
  }
}
```

#### Example without data envelope (good ✅)

```json
{
  "id": "aaa",
  "name": "Albert Einstein"
}
```

<div align="right"><a href="#top">Back to top</a></div>

## Operations

Standard CRUD operations should apply. In a situation in which certain operations are not available (read-only resources might not be available to update/delete for example), the service should return a `405 Method Not Allowed` response.

### Create

`POST /{resource:plural}`

A `POST` request to the plural instance of a resource should return a `201 Created` response with the ID of the created resource:

```jsonc
// 201 Created
{
  "id": "aaaa-bbbb"
}
```

<div align="right"><a href="#top">Back to top</a></div>

### Read

#### Retrieving a single resource by ID

`GET /{resource:singular}/{id}`

A `GET` request to the single resource instance with an ID should return a `200 OK` response with the resource:

```jsonc
// GET /user/aaa
{
  "id": "aaa",
  "name": "Steve Smith",
  "age": 30
}
```

<div align="right"><a href="#top">Back to top</a></div>

#### Searching for/retrieving a list of resources

`GET /{resource:plural}`

A `GET` request to the plural endpoint should return a `200 OK` response with a list of the resources:

```jsonc
// GET /users
[
  {
    "id": "aaa",
    "name": "Steve Smith",
    "age": 30
  },
  {
    "id": "bbb",
    "name": "Alyssa Healy",
    "age": 32
  }
]
```

<div align="right"><a href="#top">Back to top</a></div>

### Update

When you update a resource, the service should return a partial object with updated values. It's not uncommon for a resource to have various read-only computed properties. Returning updated values allows consumers of your API to keep the local state of their application in sync without having to send a follow-up `GET` request after updating a resource.

Using the following json:

```jsonc
[
  {
    "id": "aaa",
    "name": "Ellyse Perry",
    "highScore": 200,
    "average": 73.2 // read-only property
  },
  {
    "id": "bbb",
    "name": "Meg Lanning",
    "highScore": 93,
    "average": 31.36
  }
]
```

#### Replacing a resource

`PUT /{resource:singular}/{id}`

A `PUT` request should replace the entire resource and return a `200 OK` response with the updated values:

**Request:**

```jsonc
// PUT /user/bbb
{
  "name": "Meg Lanning",
  "highScore": 100
}
```

**Response:**

```jsonc
// 200 OK - return properties that have changed
{
  "highScore": 100,
  "average": 32.32
}
```

<div align="right"><a href="#top">Back to top</a></div>

#### Partially updating a resource

`PATCH /{resource:singular}/{id}`

A `PATCH` request with a partial object of the resource should return a `200 OK` response with the updated values:

**Request:**

```jsonc
// PATCH /user/aaa
{
  "highScore": 400
}
```

**Response:**

```jsonc
// 200 OK - return properties that have changed
{
  "highScore": 400,
  "average": 89.12
}
```

**Working with complex types**

It's not uncommon for resources to have non-simple types (e.g. arrays of objects):

```jsonc
// athlete object:
{
  "id": "aaa",
  "name": "aaron finch",
  "series": [
    {
      "opponent": "india",
      "scores": [
        { "score": 100, "notOut": false },
        { "score": 32, "notOut": false },
        { "score": 242, "notOut": true }
      ]
    },
    {
      "opponent": "england",
      "scores": [
        { "score": 12, "notOut": false },
        { "score": 1, "notOut": true }
      ]
    }
  ]
}
```

To partially update arrays and nested properties without having to replace the entire array, use [RFC-6902 JSON Patch](https://www.rfc-editor.org/rfc/rfc6902).

**Example:** update the athlete's latest score and mark them as out:

```jsonc
// PATCH /athlete/aaa
[
  {
    "op": "replace",
    "path": "series/1/scores/1",
    "value": { "score": 42, "notOut": false }
  }
]
```

<div align="right"><a href="#top">Back to top</a></div>

### Delete

`DELETE /{resource:singular}/{id}`

A `DELETE` request should delete the resource and return a `204 No Content` response with an empty response body.

<div align="right"><a href="#top">Back to top</a></div>

## Errors

### Response codes

- `4xx` HTTP codes should be returned for **client** errors
- `5xx` HTTP codes should be returned for **server** serrors

See [HTTP response codes](#http-response-codes) for more information on which error code to return.

### Basic conventions

- Errors should always be returned in a consistent format
- When an error cannot be inferred from the the response body, the inferred meaning from the HTTP code should take precedence (e.g. `404 Not Found` means the requested resource could not be found)
- All detected errors should be returned in a single response

### Data envelope

Similar to above, an `errors` data envelope is unnecessary and should not be returned. The fact that the application is responding with a list of errors can be inferred from the `4xx` HTTP response.

### Error object

| Property | Type     | Required | Description                                                |
| -------- | -------- | -------- | ---------------------------------------------------------- |
| property | `string` | ❌       | The property name that is the cause of the error           |
| code     | `string` | ✔️       | `CAPS_CASE` constant error code to be used programatically |
| message  | `string` | ✔️       | Human-readable description of the error.                   |

The human readable message can change over time and should not constitute a breaking change. `code` constants should not be changed (or any changes to these should constitute a breaking change).

### Example response

```json
[
  {
    "property": "dateOfBirth",
    "code": "INVALID",
    "message": "You must be 18 years old to sign up for an account"
  },
  {
    "property": "age",
    "code": "INVALID",
    "message": "Expected number, got \"string\""
  }
]
```

`property` may be omitted if the error is not in relation to a single property. For example:

```json
[
  {
    "code": "IS_CLOSED",
    "message": "The competition has closed. Please try again next year."
  }
]
```

<div align="right"><a href="#top">Back to top</a></div>

## Filtering

Filtering should be done on property names in the query parameters. For example, a simple query where you only want to return `active` users would be achieved through `GET /users?state=active`.

Advanced filtering can be achieved with the following operators:

### Basic operators

| Operator     | Description                            | Example                             |
| ------------ | -------------------------------------- | ----------------------------------- |
| `gt`         | Greater than                           | `/users?age[gt]=21`                 |
| `gte`        | Greater than or equal to               | `/users?age[gte]=18`                |
| `lt`         | Less than                              | `/users?age[lt]=25`                 |
| `lte`        | Less than or equal to                  | `/users?age[lte]=100`               |
| `eq`         | Equal to                               | `/users?name[eq]=Albert%20Einstein` |
| `contains`   | Contains                               | `/users?name[contains]=albert`      |
| `startsWith` | Starts with                            | `/users?name[startsWith]=a`         |
| `endsWith`   | Ends with                              | `/users?name[endsWith]=ein`         |
| `in`         | Comma separated list of allowed values | `/users?hairColor[in]=brown,red`    |

<div align="right"><a href="#top">Back to top</a></div>

### Null operators

`null` is a special use case, as you may (however unlikely) want to query on the string literal `"null"` as opposed to a `null` value. Because of this, the `isNull` operator is available for use\*:

| Operator | Description     | Example                       |
| -------- | --------------- | ----------------------------- |
| `isNull` | Value is `null` | `/users?dateOfDeath[isNull]=` |

_\* where possible try to avoid the use of `null` as it is often unclear what a `null` value represents._

<div align="right"><a href="#top">Back to top</a></div>

### Not condition

Use `!=` to indicate a not condition. For example, to query all users whose name does not start with the letter a:

```
GET /users?name[startsWith]!=a
```

<div align="right"><a href="#top">Back to top</a></div>

### Case sensitivity

String operators (`contains`, `startsWith`, `endsWith`, `in`) can be prefixed with `i:` (shorthand for `insensitive:`) to indicate case insensitivity. By default, filtering is assumed to be case sensitive.

**Example:** retrieve all users whose name starts with a

```
GET /users?name[i:startsWith]=a
```

<div align="right"><a href="#top">Back to top</a></div>

### Type coercion

Values sent through query parameters should be coerced where possible into the type of the property being filtered against:

- `0`/`1` should be interpreted as `false`/`true` for `booleans`
- `true` / `false` should be interpreted as `true` / `false` for `booleans`

<div align="right"><a href="#top">Back to top</a></div>

## Sorting

Sorting should be done through the `sortBy` URL parameter in the format `[propertyName].[sortOrder]`, where `sortOrder` is either `asc` or `desc`. For example, to search for users and sort by their first name in descending order, you would use:

```bash
GET /users?sortBy=firstName.desc
```

You can sort on multiple attributes through a comma:

```bash
GET /users?sortBy=firstName.desc,lastName.asc
```

<div align="right"><a href="#top">Back to top</a></div>

## Pagination

The [HTTP Link](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Link) header should be used to return links to the paginated pages.

### Cursor based pagination

#### Request

The page cursor should be passed through a `cursor` property in the URL:

```bash
GET /users?cursor=<page-cursor>
```

#### Response

Links to the `next`, `previous` and `first` page (when returning `first` is applicable) should be returned:

```
link: <https://example.rest/users?cursor=<cursor>; rel="previous", <https://example.rest/users?cursor=<cursor>; rel="next", link: <https://example.rest/users?cursor=<cursor>; rel="first"
```

<div align="right"><a href="#top">Back to top</a></div>

### Page based pagination

#### Request

The `page` and `perPage` query parameters should be used to query specific pages. `perPage` should default to an amount relative to your use case (e.g. `25`) and have a maximum value also relative to your case (e.g. `100`).

#### Response

HTTP links to the previous, next, first and last pages should be returned. The total amount of records is not directly returned, but can be loosely inferred by the URL of the `last` page. If you need to return the total amount of records, use a custom HTTP header.

Although a link to the first page may seem unnecessary, its returned to assist consumers of your API from having to parse and generate URLs.

```
link: <https://example.rest/users?page=3; rel="previous", <https://example.rest/users?page=5; rel="next", link: <https://example.rest/users?page=1; rel="first", link: <https://example.rest/users?page=22; rel="last"
```

<div align="right"><a href="#top">Back to top</a></div>

## Expanding relations

It's fairly common for resources to have relationships between them. Often a response will contain an ID of a related resource. For example, a `transaction` may have an associated `user`. To prevent consumers from having to send multiple sequential HTTP requests to retrieve this information, you should allow them to expand those objects inline with the `expand` request parameter:

```
GET /transaction/{id}?expand=user
```

Furthermore:

- You can expand multiple relations at once by separating them with a comma
- You can expand nested relations with a period (`.`)
- Expanded relations should have a maximum depth of three levels.

With the above in mind:

- Good ✅: `/transaction/{id}?expand=user,user.profile`
- Bad ❌: `/transaction/{id}?expand=user,user.profile,user.profile.address,,user.profile.address.country`

### Example

A request to `GET /transactions/<id>` might return:

```json
{
  "id": "aaaa-bbbb",
  "title": "Sample event subscription",
  "user": "zzzz-0000"
}
```

Where `user` references the ID of the `user` who made the transaction. To automatically expand on the user details, you would request `GET /transaction/{id}?expand=user,user.profile`, which might return:

```json
{
  "id": "aaaa-bbbb",
  "title": "Sample event subscription",
  "user": {
    "id": "zzzz-0000",
    "username": "albert.einstein",
    "email": "albert.einstein@example.com",
    "profile": {
      "gender": "male",
      "about": "Physicist"
    }
  }
}
```

<div align="right"><a href="#top">Back to top</a></div>

## Partial responses

It's often the case when consuming an API you're only interested in a small part of the response. In this situation, you should be able to request only certain properties through the `fields` query parameter.

On the implementation side of things, I tend to use [JSONPath](https://github.com/json-path/JsonPath), though [XPath](https://en.wikipedia.org/wiki/XPath) is a nice alternative when working with XML.

### Basic rules (assuming JSONPath):

To simplify filtering for the user, I make the JSONPath selector prefixes `$.`/`$..` optional.

Given the following `athlete` resource:

```json
{
  "name": "Ian Thorpe",
  "nationality": "AU",
  "born": 1982,
  "olympics": [
    {
      "year": 2000,
      "location": "Sydney, Australia",
      "medals": [
        { "type": "GOLD", "sport": "swimming", "event": "400 m freestyle" },
        { "type": "GOLD", "sport": "swimming", "event": "4x100 m freestyle" },
        { "type": "GOLD", "sport": "swimming", "event": "4x200 m freestyle" },
        { "type": "SILVER", "sport": "swimming", "event": "200 m freestyle" },
        { "type": "SILVER", "sport": "swimming", "event": "4x100 m medley" }
      ]
    },
    {
      "year": 2004,
      "location": "Athens, Greece",
      "medals": [
        { "type": "GOLD", "sport": "swimming", "event": "200 m freestyle" },
        { "type": "GOLD", "sport": "swimming", "event": "400 m freestyle" },
        { "type": "SILVER", "sport": "swimming", "event": "4x200 m freestyle" },
        { "type": "BRONZE", "sport": "swimming", "event": "100 m freestyle" }
      ]
    },
    {
      "year": 2012,
      "location": "London, England",
      "medals": []
    }
  ]
}
```

The following rules apply:

| Query (`?filter=???`)                       | Result                                                          |
| ------------------------------------------- | --------------------------------------------------------------- |
| `name,nationality`                          | The athlete name and nationality                                |
| `olympics`                                  | The entire olympics object                                      |
| `name,olympics.year,olympics.location`      | The athlete name and year/location of olympics they competed in |
| `olympics[0]`                               | The first olympics                                              |
| `olympics[0:1]`                             | The first two olympics                                          |
| `olympics[-2]`                              | The second to last olympics                                     |
| `olympics[1:2]`                             | All olympics from index 1 (inclusive) until index 2 (exclusive) |
| `olympics[1:]`                              | All olympics from index 1 (inclusive) to last                   |
| `olympics[?(@.year > 2000)]`                | All olympics after the year 2000                                |
| `olympics[?(@.event =~ /^\dx\d+\sm\s.*$/)]` | All olympics matching regex for team events                     |

See [JsonPath](https://github.com/json-path/JsonPath) for more information.

<div align="right"><a href="#top">Back to top</a></div>

## Caching

Both [ETag](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/ETag) and datetime [If-Modified-Since](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/If-Modified-Since) caching should be supported.

### ETag

When a resource is retrieved over the API, an `ETag` header should be returned with a hashed value that uniquely identifies the version of the resource at that point in time.

#### Preventing write collisions

When updating resources, the `ETag` and `If-Match` header can be used to ensure only a specific version of a resource is being mutated. For example, when requesting a resource with `GET /article/{id}` the service should return an ETag header in the response:

```
ETag: "aaaa-zzzz"
```

When updating the record, an `If-Match` header can be supplied to ensure only that version of the resource is being updated:

```
PATCH /article/{id}
If-Match: "aaaa-zzzz"
Content-Type: application/json

{
  "title": "My updated title"
}
```

If the hashes don't match, it means the document has likely been updated, and the server should respond with a `412 Precondition Failed` response.

#### Reading resources

The `If-None-Match` header can be used to only retrieve the resource if the hash of the most recent version doesn't match the hash supplied:

```
GET /article/{id}
If-None-Match: "aaaa-zzzz"
```

If the hashes don't match, a `200 OK` response should be returned with the resource. If the hashes do match, a `304 Not Modified` response should be returned with an empty response.

<div align="right"><a href="#top">Back to top</a></div>

### If-Modified-Since

It's often also useful to cache resources in respect to time. This is particularly useful given all resources should have a computed `updatedAt` property with the time the resource was last modified.

A requests with an [RFC 1123](http://www.ietf.org/rfc/rfc1123.txt) timestamp in the `If-Modified-Since` request header should return a `304 Not Modified` header and empty request body if the resource was last modified _before_ that time. If the resource was modified after that time, a standard `2xx` response should be returned.

```
GET /article/{id}
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

<div align="right"><a href="#top">Back to top</a></div>

### Priority

When both a `If-Modified-Since` and an `If-Match`/`If-None-Match` header are supplied, priority should always be given to the `ETag` headers.

<div align="right"><a href="#top">Back to top</a></div>

## Versioning

Versioning strategies are very situational, and I don't think there's any one approach that can always be considered as the best option.

For simplicity, default to versioning in the URL through a `v{number}` prefix. e.g. `/v1/*`, `/v2/*`. On an individual service, I generally prefer versioning before the resource at the service level:

- `/v1/user/{id}`
- `/v1/users`
- `/v1/invoices`

If your resources are fairly independent of each other you can choose to version at the resource level, however I tend to think this can sometimes increase the complexity of an API:

- `/user/v3/{id}`
- `/users/v3`
- `/transaction/v2/{id}`

When working in a service-oriented architecture, version at the service level:

**User service:**

- `GET /user/v1/{id}` -> routes to `v1` of the `user` service
- `GET /users/v2` -> routes to `v2` of the `user` service
- `GET /users/v3/{id}/transactions` -> get `user` transactions from `v3` of the `user` service

<div align="right"><a href="#top">Back to top</a></div>

## Reserved query parameters

With all of the above in mind, the following query parameters are reserved:

- **[\[sorting\]](#sorting)** `sortBy`
- **[\[pagination\]](#pagination)** `cursor` (when using [cursor based pagination](#cursor-based-pagination))
- **[\[pagination\]](#pagination)** `page` / `perPage` (when using [page based pagination](#page-based-pagination))
- **[\[expanding relations\]](#expanding-relations)** `expand`
- **[\[partial responses\]](#partial-responses)** `fields`

Where possible, try to avoid resource property names that conflict with the above reserved keywords. If for whatever reason there is a conflict, the reserved keywords should take precedence. You can query on the conflicting resource keywords by prefixing the property with a `$` symbol:

**Example:** get all users who have an `expand` value of `true` on the paginated cursor `xxxx`:

```
GET /users?cursor=xxxx&$expand=true
```

<div align="right"><a href="#top">Back to top</a></div>
