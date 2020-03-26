---
layout: ballerina-inner-page
title: The Network in the Language
permalink: /why/the-network-in-the-language/
---

# The Network in the Language

In a microservice architecture, smaller services are built, deployed and scaled individually. These disaggregated services communicate with each other over the network forcing developers to deal with the [Fallacies of Distributed Computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) as a part of their application logic.

For decades, programming languages have treated networks simply as I/O sources. Ballerina treats the network differently by making networking concepts like client objects, services, resource functions, and listeners a part of the syntax. So you can use the language-provided constructs to write network programs that just work.

## Services
Ballerina introduces service typing where services, which work in conjunction with a listener object, can have one or more resource methods in which the application logic is implemented. The listener object provides an interface between the network and the service. It receives network messages from a remote process according to the defined protocol and translates it into calls on the resource methods of the service that has been attached to the listener object.

### Get Started
Here’s a simple Hello World service to get you started:

```ballerina
import ballerina/http;
import ballerina/log;
 
listener http:Listener helloWorldEP = new(9090);
 
service hello on helloWorldEP {
   resource function sayHello(http:Caller caller, http:Request request) {
       var result = caller->respond("Hello World!");
       if (result is error) {
           log:printError("Error in responding ", err = result);
       }
   }
}
```

The Ballerina source file can compile into an executable jar:

```bash
$ ballerina build hello.bal
Compiling source
    hello.bal
Generating executables
    Hello.jar
$ java -jar hello.jar
[ballerina/http] started HTTP/WS listener 0.0.0.0:9090
$ curl http://localhost:9090/hello/sayHello
Hello, World!
```

Ballerina services come with built-in concurrency. Every request to a resource method is handled in a separate strand (Ballerina concurrent unit), which gives implicit concurrent behavior to a service.

Some protocols supported out-of-the-box include:  

* <a href="https://ballerina.io/v1-2/learn/by-example/https-listener.html">HTTPS</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-to-websocket-upgrade.html">WebSocket Upgrade</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-1.1-to-2.0-protocol-switch.html">HTTP 2.0</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/grpc-unary-blocking.html">gRPC</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/nats-basic-client.html">NATS</a>


## Async Network Protocol
In the request-response paradigm, network communication is done by blocking calls, but blocking a thread to a network call is very expensive. That’s why other languages supported async I/O and developers have to implement async/await by using callback-based code techniques.

On the other hand, Ballerina’s request-response protocols are implicitly non-blocking and will take care of asynchronous invocation.

### Get Started
The code snippet below shows a call to a simple HTTP GET request endpoint:

```ballerina
import ballerina/http;
import ballerina/io;
 
http:Client clientEndpoint = new("http://postman-echo.com");
 
public function main() {
   var response = clientEndpoint->get("/get?test=123");
   if (response is http:Response) {
       // logic for handle response
   } else {
       io:println("Error when calling the backend: ", response.reason());
   }
}
```

The above “get” operation is seemingly a blocking operation for the developer, but internally it does an asynchronous execution using non-blocking I/O, where the current execution thread is released to the operating system to be used by others. After the I/O operation is done, the program execution automatically resumes from where it was suspended. This pattern gives the developer a much more convenient programming model than handling non-blocking I/O manually while providing maximum performance efficiency. 


## Client Objects
Client objects allow workers to send network messages that follow a certain protocol to a remote process. The remote methods of the client object correspond to distinct network messages defined by the protocol for the role played by the client object. 

### Get Started
The following sample illustrates sending out a tweet by invoking tweet remote method in the twitter client object. 

```ballerina
import ballerina/config;
import ballerina/log;
import wso2/twitter;
// Twitter package defines this type of endpoint
// that incorporates the twitter API.
// We need to initialize it with OAuth data from apps.twitter.com.
// Instead of providing this confidential data in the code
// we read it from a toml file.
twitter:Client twitterClient = new ({
   clientId: config:getAsString("clientId"),
   clientSecret: config:getAsString("clientSecret"),
   accessToken: config:getAsString("accessToken"),
   accessTokenSecret: config:getAsString("accessTokenSecret"),
   clientConfig: {}
});
public function main() {
   twitter:Status|error status = twitterClient->tweet("Hello World!");
   if (status is error) {
       log:printError("Tweet Failed", status);
   } else {
       log:printInfo("Tweeted: " + <@untainted>status.id.toString());
   }
}
```


## Sequence Diagrams
The syntax and semantics of Ballerina’s abstractions for concurrency and network interaction were designed to closely correspond to sequence diagrams. This provides a visual model that shows how the program interacts using network messages.

### Get Started
The sequence diagram below is generated from a sample Salesforce integration microservice.

![Salesforce integration microservice Ballerina sequence diagram](/img/why-pages/the-network-in-the-language-1.png)
 
To start generating a sequence diagram from your Ballerina code, download the [VSCode plugin](https://ballerina.io/learn/vscode-plugin/) and [launch the graphical editor](https://ballerina.io/v1-2/learn/vscode-plugin/graphical-editor).

<a href="https://ballerina.io/why/sequence-diagrams-for-programming">Learn More &gt;</a>


## Resiliency 

_The network is unreliable_. That’s why network programs need to be written in a way that handles failures. In some cases, an automatic retry will help recover from failures while in others failover techniques will help deliver uninterrupted service. Techniques like circuit breakers also help to prevent catastrophic cascading failure across multiple programs.

Ballerina helps developers write resilient, robust programs with out-of-the-box support for techniques such as:
* <a href="https://ballerina.io/v1-2/learn/by-example/http-circuit-breaker.html">Circuit Breaker</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-load-balancer.html">Load Balancing</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-failover.html">Failover</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-retry.html">Retry</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/http-timeout.html">Timeout</a>

### Get Started

The code snippet below shows how you can easily configure a circuit breaker to handle network-related errors in the Ballerina HTTP client object.

```ballerina
http:Client backendClientEP = new("http://localhost:8080", {
       circuitBreaker: {
           rollingWindow: {
               timeWindowInMillis: 10000,
               bucketSizeInMillis: 2000,
               requestVolumeThreshold: 0
           },
           failureThreshold: 0.2,
           resetTimeInMillis: 10000,
           statusCodes: [400, 404, 500]
       },
       timeoutInMillis: 2000
   });
```

## Error Handling

Due to the inherent unreliability of networks, errors are an expected part of network programming. That’s why in Ballerina errors are explicitly checked rather than thrown as exceptions. It’s impossible to ignore errors by design because of Ballerina’s comprehensive error handling capabilities:
* <a href="https://ballerina.io/v1-2/learn/by-example/error-handling.html">Error Handling</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/check.html">Check</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/panic.html">Panic</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/checkpanic.html">Check Panic</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/trap.html">Trap</a>
* <a href="https://ballerina.io/v1-2/learn/by-example/user-defined-error.html">User-defined Error Types</a>

### Get Started

Below is a simple example of how you can explicitly check for errors:

```ballerina 
twitter:Status|error status = twitterClient->tweet("Hello World!");
   if (status is error) {
       log:printError("Tweet Failed", status);
   } else {
       log:printInfo("Tweeted: " + <@untainted>status.id.toString());
   }
```

The `tweet` remote method can return the expected `twitter:Status` value or an error due to network unreliability. Ballerina supports union types so the `status` variable can be either `twitter:Status` or `error` type. Also the Ballerina IDE tools support type guard where it guides developers to handle errors and values correctly in the if-else block.


## Network Data Safety
Distributed systems work by sharing data between different components. Network security plays a crucial role because all these communications happen over the network. Ballerina provides built-in libraries to [implement transport-level security and cryptography to protect data](https://ballerina.io/v1-2/learn/by-example/crypto.html).
 
Identity and access management also plays a critical role in microservice-based applications. Ballerina supports out-of-the-box protection for services as well as clients by using basic-auth, OAuth and JWT. The following BBEs show how to secure services and clients by enforcing authorization.

<table>      
    <tr>
        <td style="width:200px"><strong>Service</strong></td>
        <td style="width:200px"><strong>Client</strong></td>
    </tr>
    <tr>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-service-with-basic-auth.html">Basic Auth</a></td>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-client-with-basic-auth.html">Basic Auth</a></td>
    </tr>
    <tr>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-service-with-jwt.html">JWT</a></td>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-client-with-jwt-auth.html">JWT</a></td>
    </tr>
    <tr>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-service-with-oauth2.html">OAuth2</a></td>
        <td><a href="https://ballerina.io/v1-2/learn/by-example/secured-client-with-oauth2.html">OAuth2</a></td>
    </tr>
</table>

_Ballerina ensures security by default_. Its built-in <a href="https://ballerina.io/v1-2/learn/by-example/taint-checking.html?utm_source=infoq&utm_medium=article&utm_campaign=network_in_the_language_article_infoq_feb20">taint analyzer</a> makes sure that malicious, untrusted data doesn’t propagate through the system. If untrusted data is passed to a security-sensitive parameter, a compiler error is generated. You can then redesign the program to erect a safe wall around the dangerous input.

## Observability by Default

Increasing the number of smaller services that communicate with each other means that debugging an issue will be harder. Enabling observability on these distributed services will require effort. Monitoring, logging, and distributed tracing are key methods that reveal the internal state of the system and provide observability.

Ballerina becomes fully observable by exposing itself via these three methods to various external systems. This helps with monitoring metrics such as request count and response time statistics, analyzing logs, and performing distributed tracing. For more information, follow this guide:
* <a href="https://ballerina.io/learn/how-to-observe-ballerina-code">How to Observe Ballerina Services</a>


<style>
.nav > li.cVersionItem {
    display: none !important;
}
</style>