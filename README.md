# Spring Boot Websocket Example

In our current project we want to add a service that uses websockets to push messages to our mobile applications. While the documentation on [Spring Websockets + STOMP](https://spring.io/guides/gs/messaging-stomp-websocket/) is excellent when it comes to implementing a service that is consumed by a simple web application, the [example on how to use the STOMP client](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/websocket.html#websocket-stomp-client) doesn't really align very well with the short getting started guide. Since I hit a few snags in the implementation I'm creating this example so you don't have to.

## Introduction

For the spike I'm working on now we want to have our mobile client communicate with our back end via websockets. On top of that we want to use the [STOMP](http://stomp.github.io/) messaging protocol / broker since we are likely going to need a pub/sub mechanism anyway. The main difference from the [Getting Started](https://spring.io/guides/gs/messaging-stomp-websocket/) is that we don't need the SockJS compatibility layer since we control both the server and the client.

## The service

Relative to the [Getting Started](https://spring.io/guides/gs/messaging-stomp-websocket/) there are a few key changes. First of all I prefer to use [Lombok](https://projectlombok.org/) in my projects. You'll notice that the POJO's have generated getters and setters and the @Slf4j annotation creates a Logger for us.

Secondly I have removed the SocksJS layer from the service. This is a really minor change in the [endpoint configuration](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/service/WebSocketConfig.java). Instead of this:

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/hello").withSockJS();
    }

You do this:

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/hello");
    }
    
I also added a [TimeSender](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/service/TimeSender.java) component that, unsurprisingly, broadcasts time updates every 5 seconds. It's a Spring @Scheduled task. Don't forget to enable task scheduling with @EnableScheduling in your configuration! It does this on the same '/topic/greeting' topic as the [GreetingController](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/service/GreetingController.java) does to keep the client simple.

The rest of the service is more or less the same as the one in the [Getting Started](https://spring.io/guides/gs/messaging-stomp-websocket/): Application is the main entry point, [WebSocketConfig](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/service/WebSocketConfig.java) contains the configuration and [GreetingController](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/service/GreetingController.java) handled the receiving of the client's name and greeting them. 
