# Assignment


### Technology used

- Software
    - Java 17 && Spring Boot 3.2.1
    - Maven
    - Docker (Optional)
    - Docker Compose (Optional)
### How to run?
If you have docker installed, run the following command.
It will build a jar, create a docker image from it, and then use docker-compose to launch 2 containers. One for the backend
services and the other for the aggregator application (using custom docker spring profile).
```
./run.sh
```

To stop the containers, run the following script :
```
./stop.sh
```
If you do not have docker installed, or face an issue with the script, please run the following commands :
```
./mvnw clean install
java -jar target/target/aggregator-0.0.1-SNAPSHOT.jar
```
Then you need to run the container from *xyzassessment/backend-services* for the backend service 
```
docker run -p 127.0.0.1:9090:8080/tcp xyzassessment/backend-services
```

### Solution explanation

This application is using SpringBoot and Webflux. The solution is reactive.

#### CustomClient :

I noticed that the clients used to call the downstream services (Pricing, Track and Shipments) have all in
common (timeout, input parameters type..) except the return type (respectively Double, String and List<String>. 
That's why I created a generic class CustomClient that was used to instantiate 3 beans (pricingClient, trackClient and
shipmentsClient).

#### AggregationController 
This exposes the "/aggregation" endpoint. It accepts 3 optional parameters :
- pricing
- track
- shipments

#### AggregateResult
This is the pojo used to collect the answer of the Aggregation endpoint.

#### AggregationService

This is where the aggregation and the buffering happens.
The solution uses the concept of Sink. We have 3 sinks, one for each downstream service.
Whenever we receive a call to the aggregation service, we emit the parameters to the right sink
(example : pricingParams will be emitted to the pricingSink)
We create a flux from each sink then we use the flux *.bufferTimeout(maxSize, maxDuration)* method for batching.
In fact, we call a downstream service only when we have 5 params or as soon as 5 seconds 
have passed since the last non-used param was inserted in the sink.
We then use flatmap to call the downstream service and get the results.
From the results in the flux, we create a Mono for each downstream service using Mono.Create . 
This mono will contain all the results per downstream service for the parameters passed in the aggregation request.
(We use *.takeUntil* method to wait for the result of each parameter)

In fact, for the following scenario 2 calls for a downstream service are needed :
If the pricing buffer already contains 4 elements, and we get the following request :
```
GET http://<host>:8080/aggregation?
pricing=NL,CN&track=109347263,123456891&shipments=109347263,123456891
```
A first query to the pricing downstream service will be issued with the 4 existing elements in the buffer + 
"NL" (we reach the max buffer size = 5). The remaining parameter "CN" will be sent in a second query. That's why
we need to wait till we get all the results for the parameters.