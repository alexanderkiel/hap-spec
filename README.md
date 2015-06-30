__This specification is a DRAFT.__

# HAP - Hypermedia Application Protocol

# Rationale

The Hypermedia Application Protocol is a domain generic, hypermedia protocol 
emphasising self-describing representations. HAP builds on top of 
[Transit][transit] which is a data exchange format with implementations in 
various languages. HAP brings semantics for linking, manipulation and embedding 
of resources. HAP is self-describing and offers extensible data types. 

HAP is intended to be language independent. Currently Transit is already 
available for the following languages:

* Clojure
* ClojureScript
* Java
* JS
* Python
* Ruby

A Clojure Ring middleware for HAP is available as [ring-hap][ring-hap] and a 
Clojure(Script) client for HAP as [hap-client-clj][hap-client-clj].

# Spec

## Representations

Hypermedia Application Protocol representations are defined in terms of data 
structures like maps, lists, primitive types and extension types. The transport 
format Transit supports this. The minimal HAP representation is an empty map:

```json
{}
```

The minimal representation has to be a map because HAP reserves five top-level 
map keys for its features. This keys are `:links`, `:queries`, `:forms`, 
`:ops` and `:embedded`. Other than that, HAP doesn't say anything about the 
rest of the representation.

## Transport

HAP representations are transferred through [HTTP][rfc-7230]. APIs conforming to 
HAP are [RESTful][rest] APIs. The HTTP version used in HAP is 1.1 specified in
RFC 7230 and following. HAP clients and servers MUST conform to HTTP 1.1.

## Keywords

HAP uses Transit keywords for map keys. Keywords are special strings which can
have namespaces and are cached in Transit. They are typically used as map keys, 
enums or other controlled vocabulary. In this representation, keywords are 
written with a leading colon like this: `:keyword`. In Transit itself keywords 
are encoded as strings like so: `"~:keyword"`.

Keywords can have namespaces. Some user defined keywords like link relations
have to use such namespaces to avoid clashes with other keywords. A keyword with 
namespace consists of a namespace and a name and is encodes like this:
`:namespace/name`. Namespaces can have several components which are separated by 
dots like: `:com.domain.subdomain/name`.

## Links

Links are used to make your API discoverable. They link to other resources
yielding new HAP representations. Every HAP representation has one top-level 
link map keyed under the reserved `:links` key. The link map itself keys all
links under there link relation. A HAP representation with a self link looks 
like this:

```json
{"~:links": {"~:self": {"~:href": "~rhttp://..."}}}
```

Every link is a map. There are the following reserved keys:

* :href - the URI of the link.
* :label - a human readable label of the resource of the link target (optional)

[URIs][rfc-3986] in HAP are always specified using the Transit semantic type 
`uri`. Relative URIs are allowed and have to be resolved against the 
[effective request URI][rfc-7230-5.5].

### Link Relation Types

Link relation types follow the [RFC 5988][rfc-5988]. They are expressed as 
keywords. 

Registered link relation types are keywords without a namespace where the name 
of the keyword is character-by-character equal to the token of the link relation 
type.

The following registered link relation types are used in HAP according there
established semantics:

* :self - MUST be a URI of the resource which produced the HAP representation.
          The URI should be canonical. The self link relation is listed at 
          [IANA][iana-link-rels] and defined in [RFC 4287][rfc-4287]. It acts
          as base URI of relative URIs.

* :next - TODO

* :prev - TODO

Extension link relation types are encoded as keywords with a namespace. 

### Specifying more than one Link per Link Relation

It is possible to specify more than one link per link relation. A common use
case it to a link to a collection of items like line items in an order. More 
than one link can be specified by using an array of links like so:

```json
{"~:links":
 {"~:line-items":
  [{"~:href": "~rhttp://..."},
   {"~:href": "~rhttp://..."}]}}
```

## Queries

Queries are used to describe which query params a resource accepts. Queries are
similar to HTML forms which use HTTP GET. One example of a query is simple 
filtering of the items in a list-like resource. Resources providing filtering 
will contain a query like this:

```json
{"~:queries": 
  {"~:filter": 
    {"~:href": "~rhttp://...",
     "~:params": {"~:filter": {"~:type": "~$Str"}}}}}
```

### Executing Queries

Queries are executed by issuing an HTTP GET request against a URI which is build
the following way. First the URI from `:href` has to be resolved against the 
[effective request URI][rfc-7230-5.5] of the representation containing the query.
Second each parameter is encoded as query param of the URI were parameter names 
are encoded as strings and the values itself are encoded using non-verbose 
Transit JSON encoding.

Resources accepting queries have to return regular representations either 
directly or through redirection.

## Forms

In difference to links and queries, forms describe how new resources can be 
created. In that sense, forms are equivalent to HTML forms using HTTP POST. 
Forms are important in order to make HAP representations self describing. With 
forms users of an API can identify possible resource manipulations without the 
need of any out-of-band representationation.

A HAP representation with a form looks like this:

```json
{"~:forms": {"~:name": {"~:href": "~rhttp://...",
                        "~:title": "..."}}}
```

The `:href` key carries an URI which is encoded according the Transit spec and 
points to the resource handling the form. Relative URIs are allowed and have 
to be resolved against the [effective request URI][rfc-7230-5.5].

The `:title` key specifies a human readable title which is optional.

Form have parameters which can convey values of arbitrary semantic types 
including composite types. Required and optional parameters are specified in a 
`:params` map:

```json
{"~:href": "~rhttp://...",
 "~:params": {"~:content": {"~:type": "~$Str"},
              "~:due": {"~:type": "~$Inst"}}}
```

Params have names which are keywords. Each param is a map itself were the 
following keys are reserved:

* :type - the schema describing the param
* :optional - a boolean value which defaults to false (optional)
* :desc - a human readable description of the param (optional)

Types are currently specified in form of [Prismatic Schema][schema] expressions,
but this might change in the future because Prismatic Schema is only 
implemented in Clojure and ClojureScript right now. Apart from that, validation
of form parameters is fully optional. HAP representations are not typed at all
 and so are form param values. A server might use the schema specified for an 
form param to validate its value but this is not a requirement. At the end 
following [Postel's Law][postel] is preferred over overly strict validation of 
inputs.

### Example Schemas

| Schema | Transit Semantic Type | Notes |
|:-------|:----------------------|:------|
|Str     |string                 ||
|Inst    |point in time          |a date/time according RFC 3339 without offsets|
|(enum :a :b)|keyword            |an enum with the two allowed keyword values :a and :b|

### Submitting Forms

Forms are exclusively used to create new resources. Clients have to use HTTP
POST to issue a form request. The request payload is a Transit encoded map of
parameter names to the values supplied. The resource has to respond with status
code 201 and a location header containing the URI of the created resource.
Client can than fetch the newly created resource if they like.

### Example Form

This example is a form which lets you create new todo items. There are two 
prams specified: `:content` and `:due` where `:content` has the type `Str`
and `:due` the type `Inst`.

```json
{"~:href": "~r/todos",
 "~:title": "Create new ToDo Item",
 "~:params": {"~:content": {"~:type": "~$Str"},
              "~:due": {"~:type": "~$Inst"}}}
```

The representation to post looks like this:

```json
{"~:content": "Buy milk", 
 "~:due": "~t2016-04-12T23:20:50.52Z"} 
```

## Embeddable Representations

Representations of resources can be embedded instead of only linked. Embedding
representations is a performance optimisation and equivalent to link to there
resource. One common example are line items of an order. 

```json
{"~:embedded":
  {"~:line-items":
    [{"~:amount": 1,
      "~:links": {"~:self": {"~:href": "~rhttp://..."},
                  "~:product": {"~:href": "~rhttp://..."}}},
     {"~:amount": 2,
      "~:links": {"~:self": {"~:href": "~rhttp://..."},
                  "~:product": {"~:href": "~rhttp://..."}}}]}}
```

Embedded representations MUST contain a `:self` link. So there has to be always 
a resource providing the embedded representation. Embedded representations
MUST NOT be used for simple composite values which are always possible.

It is not necessary that an embedded representation equals the representation
conveyed by its resource. A common use case is to provide a subset of the data
as embedded representation were the full representation can be obtained by 
following the `:self` link.

## Generic Operations

Apart from application specific queries and forms describes earlier, HAP 
provides semantics for the following generic operations.

### Full Resource Updates

A full resource update is done by putting a representation which should 
represent the new state of the resource to the URI of the resource. A
[conditional request][rfc-7232] SHOULD be used to issue the PUT. Successful 
responses to update requests have the status code 
[204 No Content][rfc-7231-6.3.5] and contain no body.

Partial updates are not possible right now. They may be added later.

Representations of resources were full updates are used should not contain
embeddable representations. When embeddable representations are used, the ETag
has to change whenever one of the embedded representations change. Otherwise
caching of such representations would not work as intended. On the other hand
the ETag is used to detect state changes of the resource itself to prevent
clients from overwriting previous changes. However if the ETag changes only
because an embedded representation changed, the update request of a client will
be refused without a good reason. So ETags of updateable resources should only
reflect the resource state and so embeddable representations are not suitable.

Full resource updates have several advantages. One advantage is the possibility 
to use a conditional request which prevents one from the "lost update" problem.
Another advantage is cache invalidation. HTTP caches understand PUT operations
on resources and invalidate the cached representation accordingly. 

### Resource Deletion

A resource can be deleted by issuing a HTTP DELETE request to the resource URI.
Successful responses to delete requests have the status code 
204 No Content and contain no body.

### Operation Announcement

HAP representations contain a vector under the `:ops` key. This vector holds
two keywords `:update` and `:delete` depending whether the operation is 
implemented on the resource delivering the representation.

## Related Work

### HTML

HTML - Hypertext Markup Language is a media type for presenting textual 
representations to humans. A browser is used to drive the interaction between 
the human and the server providing HTML representations. The most important 
point is that the browser is fully generic, which means it does not depend on 
the server or say the application with provides the HTML representations. It's 
the strength of HTML which allows a rich interaction between a human and a 
server using a generic browser.

HAP builds on the foundations layed out by HTML. HAP takes the good stuff from
HTML - the links and forms - and removes just the presentation artifacts,
concentrating on pure data.

### HAL

HAL - Hypertext Application Language ist just the same as HAP but without 
queries and forms. Furthermore HAL uses JSON which lacks extensible value types. 
HAL on top of Transit would be very near to HAP.

[iana-link-rels]: <http://www.iana.org/assignments/link-relations/link-relations.xhtml>
[rfc-3339]: <http://tools.ietf.org/html/rfc3339>
[rfc-3986]: <http://tools.ietf.org/html/rfc3986>
[rfc-4287]: <http://tools.ietf.org/html/rfc4287>
[rfc-5023]: <http://tools.ietf.org/html/rfc5023>
[rfc-5988]: <http://tools.ietf.org/html/rfc5988>
[rfc-6570]: <http://tools.ietf.org/html/rfc6570>
[rfc-7230]: <https://tools.ietf.org/html/rfc7230>
[rfc-7230-5.5]: <https://tools.ietf.org/html/rfc7230#section-5.5>
[rfc-7231]: <https://tools.ietf.org/html/rfc7231>
[rfc-7231-6.3.5]: <https://tools.ietf.org/html/rfc7231#section-6.3.5>
[rfc-7232]: <https://tools.ietf.org/html/rfc7232>
[cj]: <http://amundsen.com/media-types/collection/>
[profile]: <http://tools.ietf.org/html/draft-wilde-profile-link-04>
[discovery]: <https://developers.google.com/discovery/>
[json]: <http://json.org/>
[json-ld]: <http://json-ld.org/>
[siren]: <https://github.com/kevinswiber/siren>
[curie]: <http://www.w3.org/TR/curie/>
[hal]: <http://stateless.co/hal_specification.html>
[draft-hal]: <http://tools.ietf.org/html/draft-kelly-json-hal-06>
[transit]: <https://github.com/cognitect/transit-format>
[schema]: <https://github.com/Prismatic/schema>
[postel]: <http://en.wikipedia.org/wiki/Robustness_principle>
[transit-i16]: <https://github.com/cognitect/transit-format/issues/16>
[ring-hap]: <https://github.com/alexanderkiel/ring-hap>
[hap-client-clj]: <https://github.com/alexanderkiel/hap-client-clj>
[rest]: <http://en.wikipedia.org/wiki/Representational_state_transfer>
[fielding-1]: <http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven>