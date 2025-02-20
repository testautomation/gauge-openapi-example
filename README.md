# gauge-openapi-example

[![Gauge Badge](https://gauge.org/Gauge_Badge.svg)](https://gauge.org) 
[![Gauge](https://github.com/agilepathway/gauge-openapi-example/workflows/Gauge%20specs/badge.svg)](https://github.com/agilepathway/gauge-openapi-example/actions?query=workflow%3A%22Gauge+specs%22+branch%3Amaster)
[![reviewdog](https://github.com/agilepathway/gauge-openapi-example/workflows/reviewdog/badge.svg)](https://github.com/agilepathway/gauge-openapi-example/actions?query=workflow%3Areviewdog+event%3Apush+branch%3Amaster)
[![License](https://img.shields.io/github/license/agilepathway/gauge-openapi-example?color=blue)](LICENSE)
[<img src="https://github.com/agilepathway/gauge-openapi-example/wiki/images/openapi.png" width="87">](./openapi.yaml)[![OpenAPI Validator](https://validator.swagger.io/validator?url=https://raw.githubusercontent.com/agilepathway/gauge-openapi-example/master/openapi.yaml)](./openapi.yaml)

Example in Python of how [Gauge](https://gauge.org/) and [OpenAPI](https://www.openapis.org/about) play nicely
together to produce [living documentation](https://www.infoq.com/articles/book-review-living-documentation/) for APIs.

NB There is also a separate [Java example repository](https://github.com/agilepathway/java-gauge-openapi-example),
demonstrating the same workflow but using Java as the test implementation language instead of Python.

___
* [Example workflow](#example-workflow)
* [Benefits of this approach](#benefits-of-this-approach)
* [Running the spec](#running-the-spec)
  * [Prerequisites](#prerequisites)
  * [Run the spec against the mock server](#run-the-spec-against-the-mock-server)
  * [Run the spec against the real server](#run-the-spec-against-the-real-server)
  * [Run the spec using Prism as a validation proxy against the real server](#run-the-spec-using-prism-as-a-validation-proxy-against-the-real-server)
* [Notes](#notes)
___

## Example workflow

1. Have a collaborative story refinement session to come up with 
   [specification examples](https://gojko.net/2008/11/04/specifying-with-examples/), using 
   [example mapping](https://cucumber.io/blog/bdd/example-mapping-introduction/) for instance

2. [Write up the specification examples in Gauge](https://docs.gauge.org/writing-specifications.html).

   Using the example from this repo, we'd have the following Gauge spec written at this point:

   ```
    # Pet store availability

    ## Customers can see which pets are available in the pet store

    * There is a pet named "doggie" available in the pet store
   ```

   We don't write the underlying implementation for this Gauge spec yet, that will come below.

3. Now, let's say that implementing this feature requires a new [REST API microservice](https://microservices.io/patterns/microservices.html).

   Create an OpenAPI specification to describe our new API, e.g. the [`openapi.yaml`](./openapi.yaml) in this repo.

   (The OpenAPI specification file can be YAML or JSON)
   
4. Even though we don't have an implementation for our OpenAPI spec yet, we already have all we need to go ahead and implement the Gauge spec.
   1. Generate an [SDK client](https://nordicapis.com/what-is-the-difference-between-an-api-and-an-sdk/) for our OpenAPI spec.
      One of the really nice things about OpenAPI is that we can generate client and server code just from the spec.
      We will use [OpenAPI Generator](https://openapi-generator.tech) to generate our client SDK code:
      1. [Install OpenAPI Generator](https://openapi-generator.tech/docs/installation)
      2. Generate the client SDK code, e.g:

         `openapi-generator-cli generate -i openapi.yaml -g python -o ./python-client-generated`

         (we use Python in our example, but you can generate code in many other languages too)
      3. Install our new Python client SDK library:

         `cd python-client-generated/ && sudo python setup.py install && cd ../`
         
    2. Now we have our Python client SDK, we can go ahead and implement the underlying code for our Gauge spec:
         
       ```
       from getgauge.python import step
       import openapi_client
       from openapi_client.api import pet_api
       import os
       
       
       @step("There is a pet named <pet_name> available in the pet store")
       def there_is_an_available_pet_named(pet_name):
           with openapi_client.ApiClient(configuration()) as api_client:
               api_instance = pet_api.PetApi(api_client)
               available_pets = api_instance.find_pets_by_status(["available"])
               print(available_pets)
               assert any(pet.name == pet_name for pet in available_pets)
       
       
       def configuration():
           openapi_host = os.environ.get("OPENAPI_HOST")
           if openapi_host is None:
               configuration = openapi_client.Configuration()
           else:
               configuration = openapi_client.Configuration(host=openapi_host)
       
           configuration.access_token = "YOUR_ACCESS_TOKEN"
           return configuration
       ```

       We did not have to write much code at all, as the Python client SDK provides all
       the boilerplate for us.
    
5. If we ran the Gauge spec now it would fail, because there is no implementation of the OpenAPI spec for the Python Client SDK to communicate with. Enter Prism.

   [Prism](https://stoplight.io/prism) is a mock server that effortlessly serves example
   responses based on an OpenAPI spec.
   1. [Install Prism](https://meta.stoplight.io/docs/prism/docs/getting-started/01-installation.md)
   3. Setup a Gauge [environment variable](https://docs.gauge.org/configuration.html#using-environments-in-a-gauge-project) to point our Gauge spec implementation at the Prism
   mock server that we just started:

      Create an `env/mock/openapi.properties` file and also an
      `env/validation-proxy/openapi.properties` file, both with this content: 
      
      `OPENAPI_HOST = http://127.0.0.1:4010`

6. Now we can run our Gauge spec against our mock environment, and it will pass :-)
   - `prism mock openapi.yaml --errors`
   - `gauge run --env mock specs`

7. We can now go ahead and implement the API, based on our OpenAPI spec of course.

   When we have done so, we can run our Gauge spec against it too, without any modification:

   - `gauge run specs`

   Even better, we can [use Prism as a validation proxy against the real server](https://meta.stoplight.io/docs/prism/docs/guides/03-validation-proxy.md), which verifies
   that the implementation is fully compliant with the OpenAPI spec:

   - `prism proxy openapi.yaml https://petstore.swagger.io/v2 --errors`
   - `gauge run --env validation-proxy specs`

## Benefits of this approach

1. Collaborative - consumers, solution architects, developers, testers, analysts, Product Owner all have a natural interest in being involved.  This is a great silo breaker.
2. [Shift Left](https://devops.com/devops-shift-left-avoid-failure/) - enables testing of APIs before implementation has started
3. Speeds up and improves service consumer and provider integration: provides API consumers with a working mocked example of the API that they can integrate straightaway.
4. [Design-first APIs](https://tyk.io/moving-api-design-first-agile-world/)
   - [Development teams can work in parallel](https://swagger.io/resources/articles/adopting-an-api-first-approach/#development-teams-can-work-in-parallel--3)
   - [Reduces the cost of developing apps](https://swagger.io/resources/articles/adopting-an-api-first-approach/#reduces-the-cost-of-developing-apps-4)
   - [Increases the speed to market](https://swagger.io/resources/articles/adopting-an-api-first-approach/#increases-the-speed-to-market-5)
   - [Ensures good developer experiences](https://swagger.io/resources/articles/adopting-an-api-first-approach/#ensures-good-developer-experiences-6)
   - [Reduces the risk of failure](https://swagger.io/resources/articles/adopting-an-api-first-approach/#reduces-the-risk-of-failure-7)
5. [Specification by Example](https://gojko.net/2008/11/04/specifying-with-examples/)
   - Shared understanding between all parties
   - [Living documentation](https://www.infoq.com/articles/book-review-living-documentation/), providing a single source of truth. This API documentation stays up to date because it is executable, and is only written in one place (rather than analysts, developers and testers all writing their own separate documentation.)
6. API [black box testing](https://resources.whitesourcesoftware.com/blog-whitesource/black-box-testing)
   - provides great test coverage
   - decoupled from implementation, so does not get in the way of implementation
7. [Consumer-Driven-Contract-Testing](https://meta.stoplight.io/docs/prism/docs/guides/03-validation-proxy.md#end-to-end-contract-testing)
   - allows service design to be driven by the verified needs of consumers
   - [ensures that consumer test stubs stay in sync with the implementation](https://meta.stoplight.io/docs/prism/docs/guides/03-validation-proxy.md#assisting-api-consumer-integration)
8. Enables different languages to be used easily - can choose Python for the client SDK and Java for the server implementation, for instance

## Running the spec
### Prerequisites
- [Install OpenAPI Generator](https://openapi-generator.tech/docs/installation)
- Generate the Python client SDK code:

  `openapi-generator-cli generate -i openapi.yaml -g python -o ./python-client-generated`

- Install the generated Python client SDK code:

  `cd python-client-generated/ && sudo python setup.py install && cd ../`

- [Install Prism](https://meta.stoplight.io/docs/prism/docs/getting-started/01-installation.md)
  
### Run the spec against the mock server
- `prism mock openapi.yaml --errors`
- `gauge run --env mock specs`

### Run the spec against the real server
- `gauge run specs`

### Run the spec using Prism as a validation proxy against the real server
- `prism proxy openapi.yaml https://petstore.swagger.io/v2 --errors`
- `gauge run --env validation-proxy specs`


## Notes

- This example uses [Gauge](https://gauge.org/), but other natural-language specification tools 
(e.g. [Cucumber](https://cucumber.io/), [SpecFlow](https://specflow.org/) etc) would be fine too.
