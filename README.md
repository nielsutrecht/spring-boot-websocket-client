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

## The JavaScript client

The only real change made on the [JS client side](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/resources/static/app.js) was removing the SockJS compatibility layer. I removed the script tag loading the sockjs.js source and changed the bit where you create the client from this:

    var socket = new SockJS('/gs-guide-websocket');
    stompClient = Stomp.over(socket);

To this:

    stompClient = Stomp.client('ws://localhost:8080/hello');
    
## The Java client

The Java STOMP client was a bit less straightforward. It had me stumped (stomped?) for a while. Although I could with it to the service and send messages, it would not allow me to receive any. Not just that; I wasn't getting any errors either. Why? Well what the documentation unfortunately does not tell you is that the StompSessionHandlerAdapter you tend to extend has an [empty implementation of handleException()](http://grepcode.com/file/repo1.maven.org/maven2/org.springframework/spring-messaging/4.2.0.RELEASE/org/springframework/messaging/simp/stomp/StompSessionHandlerAdapter.java#57). So whenever something goes wrong it just swallows the exception. Great design guys; only took me like 2 hours to figure out. So the first order of business was to add proper exception handling to [MySessionHandler](https://github.com/nielsutrecht/spring-boot-websocket-client/blob/master/src/main/java/com/nibado/example/websocket/client/MySessionHandler.java). 

    @Override
    public void handleException(StompSession session, StompCommand command, StompHeaders headers, byte[] payload, Throwable exception) {
        exception.printStackTrace();
    }

Why something similar isn't the default is beyond me. Note: you can't simply retrow the exception here, it will disappear. 

So now that we are actually seeing exceptions I found out I was doing something wrong:

    org.springframework.messaging.converter.MessageConversionException: No suitable converter, payloadType=class com.nibado.example.websocket.service.Greeting, handlerType=class com.nibado.example.websocket.client.MySessionHandler
	at org.springframework.messaging.simp.stomp.DefaultStompSession.invokeHandler(DefaultStompSession.java:443)
    
I went through different permutations of removing the StringMessageConverter, returning the Type of String inside session handler, etc. The problem was both a lack of understanding on my part and an issue with the documentation. First of all; the STOMP 'frames' you receive will have a mime type. It's not really easy to spot but the service implementation, because we are returning Greeting object is using Jackson to serialize these objects to JSON. This will also result in those frames getting the application/json mimetype. The StringMessageConverter will **only** work on text/plain messages.

The second part that was confusing was the documentation. It claims that if Jackson is on the classpath it will use a Jackson message convertor. Well, I don't know how it tries to figure this out but it didn't happen automatically, this is why I needed to add it myself:

    stompClient.setMessageConverter(new MappingJackson2MessageConverter());
    
That got my basic [client](https://github.com/nielsutrecht/spring-boot-websocket-client/tree/master/src/main/java/com/nibado/example/websocket/client) working: the ServiceClient configures the connection and adds a handler, MySessionHandler implements a handler that sends a message to the WS service and then subscribes to the 'topic/greeting' topic. 

# Conclusion

I hope this example is useful or at least might save you some time. If you have any questions or feedback feel free to raise an issue in this repository!
