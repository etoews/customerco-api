# CustomerCo API

An exercise in creating an API using RAML for the fictional company CustomerCo.

* [RAML Spec and API documentation](#raml-spec-and-api-documentation)
* [Use cases](#use-cases)
* [Design decisions](#design-decisions)
* [Future design work](#future-design-work)
* [Aside: An API workshop](#aside-an-api-workshop)

## RAML Spec and API documentation

The API RAML spec can be found in [api.raml](api.raml) and the MuleSoft hosted API Reference can be found in the [public portal]().

## Use cases

1. A periodic sync of customers using the API.
  * Supported by the design decision [Cache control](#cache-control).
  * Supported by the design decision [Batch PUT /customers](#batch-put-customers).

1. A mobile app that allows retrieval and update of customers details.
  * Mobile apps can be particularly sensitive to chatty APIs.
    * Chattiness is reduced by the design decisions
      * [Cache control](#cache-control)
      * [Batch PUT /customers](#batch-put-customers)
      * [Searching, sorting, and pagination](#searching-sorting-and-pagination)
    * Chattiness could also be further reduced by a batch POST /customers operation if that was a requirement.
  * Mobile apps can be particularly sensitive to APIs that required large data transfers.
    * This API hasn't been profiled so it's currently unknown if it requires large data transfers.
    * If it's determined that large amounts of data are being transferred, I would look at enabling gzip compression (see [Future design work](#future-design-work)).
    * gzip compression hasn't been enabled initially as I believe that would be a premature optimization.
  * Prevent the use of the header Connection:Keep-Alive to avoid maintaining open connections and draining the battery.

1. Simple extension of the API to support future resources such as orders and products.
  * Supported by the design decision [RAML 1.0 or 0.8?](#raml-10-or-08).
  * RAML traits are already in use in this API and can be reused by new resources make them consistent with existing resources.
  * Introduce the use of RAML resourceTypes to make common types consistent throughout the API.
  * Add link relations between related resources such as Customers and Orders. For example, in a Customer resource JSON representation.

    ```json
    "links": [
      {
        "href": "https://customerco.com/api/v1beta1/customers/980d5a09-1b94-424e-85cc-be94a700877c/orders",
        "rel": "related"
      }
    ]
    ```

## Design decisions

Below is commentary on some of the interesting design decisions made while writing this API.

### Conform to established API design guidelines or create a bespoke API?
* Everyone agreeing on established API design guidelines can prevent a lot of bikeshedding on technicalities and put the focus on delivering customer value instead.
* If a company has established API design guidelines, it's best to conform to those to the extent that they are applicable to this particular API.
* In the absence of company established design guidelines, it's worthwhile to research if public guidelines like [JSON API](http://jsonapi.org/), external company guidelines like the [Microsoft REST API Guidelines](https://github.com/Microsoft/api-guidelines), or open source project guidelines like the [Kubernetes API conventions ](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md) are applicable to this particular API.
* Decision: Bespoke API
 * Normally I'd look to use established guidelines but the spirit of this exercise is to "start a conversation about light-weight design" and that's best done by creating a bespoke API.

### RAML 1.0 or 0.8?
* RAML 0.8 has been around for a number of years and there's a fair amount of documentation and examples for it.
* RAML 1.0 went GA on May 16, 2016 but it's still a bit difficult to find docs and examples for it.
* After some digging I found this [raml1.0 branch](https://github.com/raml-org/raml-tutorial/tree/raml1.0) of the [raml-tutorial](https://github.com/raml-org/raml-tutorial) and all of the [raml-examples](https://github.com/raml-org/raml-examples) are in 1.0.
* There are some really nice features in 1.0 like libraries, improved examples, and data-typing that make it easier to have a nicely documented, factored, and consistent API.
* Decision: RAML 1.0

### UUID or integer IDs?
* [Primary Keys: IDs versus GUIDs](https://blog.codinghorror.com/primary-keys-ids-versus-guids/) and [Advantages and disadvantages of GUID / UUID database keys](https://stackoverflow.com/questions/45399/advantages-and-disadvantages-of-guid-uuid-database-keys)
* I also like that UUIDs are basically impossible to guess.
* Decision: UUID

### Address format
* Are users of this API based in a particular region, country, continent, or global?
* Address formats can be hairy beasts. Best not to make them more complicated than absolutely necessary.
* Side note: Considering there are multiple addresses per customer I discovered a need for an address label and a default address.
* Decision: A basic address format

### Full PUT, partial PUT, or PATCH for updates?
* Full PUT is simple to implement on client/server but the request body size is bigger, however this can be offset by using gzip encoding.
* Partial PUT is slightly more complicated to implement on client/server due to dirty/null/empty field handling but request body size is smaller.
* PATCH is significantly more complicated to implement on client/server. A PATCHing scheme must be chose and carefully implemented but request body size is minimal.
* Decision: Full PUT
 * This is a simple API and the additional complexity of partial PUT or PATCH is unwarranted.

### Batch PUT /customers
* To enable the use cases of "periodic sync of customers using the API" and a "mobile app that allows retrieval and update of customers details" there is an operation to batch PUT /customers.
* This operation does not replace the entire customer collection, rather only those customers included in the request body will be updated.
* This allows users to efficiently update/sync only the customers that have changed.
* Decision: Enable batch updates of customers

### Cache control
* To enable the use cases of "periodic sync of customers using the API" and a "mobile app that allows retrieval and update of customers details" the GET /customers operation implements cache control.
* This is specified and documented in [traits](traits.raml) in the cacheable section.
* Decision: Enable cache control

### Searching, sorting, and pagination
* To enable the use case of a "mobile app that allows retrieval" the GET /customers operation implements searching, sorting, and pagination.
* This allows the user to quickly find the resources they're looking for and minimize data transfer.
* This is specified and documented in [traits](traits.raml) in the searchable, sortable, and pageable sections.
* Decision: Enable searching, sorting, and pagination

### Versioning
* There are a number of ways to version an API: URI versioning, query string versioning, header versioning, media type versioning, or even no versioning at all.
* I've adopted the [URI versioning in use by the Kubernetes API](https://kubernetes.io/docs/reference/api-overview/#api-versioning).
    > The version is set at the API level rather than at the resource or field level to ensure that the API presents a clear, consistent view of system resources and behavior, and to enable controlling access to end-of-life and/or experimental APIs.
* Considering this API is very early in its lifecycle, marking it as beta is appropriate in my opinion.
* Whether or not I would actually put this in front of a paying customer, depends on if that customer is comfortable with the word "beta". Some customer may not perceive the value of a beta period for an API and may require some convincing that a beta period can be beneficial for everyone involved.
* Decision: URI versioning

### Errors
* Rich error messages are one of the most crucial and most overlooked aspects of good API design.
* Good error messages should include possible remediation steps and links to helpful resources like documentation and support.
* This work is based on my work on the [Error spec for OpenStack] (http://specs.openstack.org/openstack/api-wg/guidelines/errors.html), which was largely informed by the [Error spec for JSON API](http://jsonapi.org/format/#errors)
* This is specified and documented in [types]types.raml) in the Error type.
* Decision: Enable rich error messages

### Link Relations
* Help developers discover related resources by providing links.
* This work follows the [Link Description Object spec](http://json-schema.org/latest/json-schema-hypermedia.html#rfc.section.6) and [Link Relation Types spec](http://www.iana.org/assignments/link-relations/link-relations.xhtml).
* This is specified and documented in [types](types.raml) in the Link type.
* Decision: Enable link relations

## Future design work

The above design decisions are a good start on the design of this API. However, there are still many things to consider for a production ready API.

* Use of server certificates for TLS.
* [Security scheme](https://github.com/raml-org/raml-spec/blob/master/versions/raml-10/raml-10.md#security-schemes), [Authentication method](https://docs.mulesoft.com/anypoint-connector-devkit/v/3.8/authentication-methods)), and the use of the HTTP status code 401.
* Rate limiting and the use of the header Retry-After.
* Enabling gzip compression and the use of the header Content-Encoding (see [Using the GZIP Compress and Uncompress Transformer With MuleSoft] (https://dzone.com/articles/gzip-compress-and-uncompress-transformer-with-mule)).
* Cross-Origin Resource Sharing (see [An Authoritative Guide to CORS (Cross-Origin Resource Sharing) for REST APIs](https://www.programmableweb.com/news/authoritative-guide-to-cors-cross-origin-resource-sharing-rest-apis/how-to/2017/03/08)).
* Determining this API's approach to backwards compatibility (see [Google Cloud Platform's spec]( https://cloud.google.com/apis/design/compatibility)).
* Determining what clients will be supported. For example, software development kits (SDKs) written in Python, Java, Swift, Go, C#, Node.js, etc., command line interfaces (CLIs), web user interfaces, and mobile app user interfaces (UIs).

## Aside: An API workshop

In 2015 I co-authored and co-instructed an API workshop on REST called "A RESTful Adventure" to walk people through REST fundamentals, the design of an API, and the implementation of that API.

* [Source code](https://github.com/everett-toews/a-restful-adventure)
* [Presentation](http://everett-toews.github.io/a-restful-adventure/presentation/)
