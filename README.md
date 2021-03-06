# Weld Vert.x Extensions

The primary purpose of `weld-vertx` is to bring the CDI programming model into the Vert.x ecosystem, i.e. to extend the Vert.x tool-kit for building reactive applications on the JVM. Right now, there are two artifacts available - `weld-vertx-core` and `weld-vertx-web`.

## weld-vertx-core

```xml
<dependency>
  <groupId>org.jboss.weld.vertx</groupId>
  <artifactId>weld-vertx-core</artifactId>
  <version>${version.weld-vertx}</version>
</dependency>
```

### CDI observers as Vert.x message consumers

Vert.x makes use of a light-weight distributed messaging system to allow application components to communicate in a loosely coupled way. `weld-vertx-core` allows to automatically register certain observer methods as Vert.x message consumers and also to inject relevant `io.vertx.core.Vertx` and `io.vertx.core.Context` instances into beans.

A simple echo message consumer could look like this:

```java
import org.jboss.weld.vertx.VertxConsumer;
import org.jboss.weld.vertx.VertxEvent;

class Foo {
    public void echoConsumer(@Observes @VertxConsumer("test.echo.address") VertxEvent event) {
        event.setReply(event.getMessageBody());
    }
}
```
* `@VertxConsumer` - a qualifier used to specify the address the consumer will be registered to: test.echo.address
* `VertxEvent` - a wrapper of a Vert.x message

Since we’re working with a regular observer method, additional parameters may be declared (next to the event parameter). These parameters are injection points. So it’s easy to declare a message consumer dependencies:

```java
public void consumerWithDependencies(@Observes @VertxConsumer("test.dependencies.address") VertxEvent event, CoolService coolService, StatsService statsService) {
    coolService.process(event.getMessageBody());
    statsService.log(event);
}
```
**NOTE**: If you inject a dependent bean, it will be destroyed when the invocation completes.

Last but not least - an observer may also send/publish messages using the Vert.x event bus:

```java
public void consumerStrikesBack(@Observes @VertxConsumer("test.publish.address") VertxEvent event) {
    event.messageTo("test.huhu.address").publish("huhu");
}
```

#### How does it work?

The central point of integration is the `org.jboss.weld.vertx.WeldVerticle`. This Verticle starts Weld SE container and automatically registers `org.jboss.weld.vertx.VertxExtension` to process all observer methods and detect observers which should become message consumers. Then a special handler is registered for each address to bridge the event bus to the CDI world. Handlers use `Vertx.executeBlocking()` since we expect the code to be blocking. Later on, whenever a new message is delivered to the handler, `Event.fire()` is used to notify all relevant observers.

### CDI-powered Verticles

It's also possible to deploy Verticles produced/injected by Weld, e.g.:

```java
@Dependent
class MyBeanVerticle extends AbstractVerticle {

     @Inject
     Service service;

     @Override
     public void start() throws Exception {
         vertx.eventBus().consumer("my.address").handler(m -> m.reply(service.process(m.body())));
     }
}

class MyApp {
     public static void main(String[] args) {
         final Vertx vertx = Vertx.vertx();
         final WeldVerticle weldVerticle = new WeldVerticle();
         vertx.deployVerticle(weldVerticle, result -> {
             if (result.succeeded()) {
                 // Deploy Verticle instance produced by Weld
                 vertx.deployVerticle(weldVerticle.container().select(MyBeanVerticle.class).get());
             }
         });
     }
}
```



## weld-vertx-web

```xml
<dependency>
  <groupId>org.jboss.weld.vertx</groupId>
  <artifactId>weld-vertx-web</artifactId>
  <version>${version.weld-vertx}</version>
</dependency>
```

### Define route in a declarative way

`weld-vertx-web` extends `weld-vertx-core` and `vertx-web` functionality and allows to automatically register `Route` handlers discovered during container initialization. In other words, it's possible to configure a `Route` in a declarative way:

```java
import javax.inject.Inject;

import org.jboss.weld.vertx.web.WebRoute;

import io.vertx.core.Handler;
import io.vertx.ext.web.RoutingContext;

@WebRoute("/hello")
public class HelloHandler implements Handler<RoutingContext> {

    @Inject
    SayHelloService service;

    @Override
    public void handle(RoutingContext ctx) {
        ctx.response().setStatusCode(200).end(service.hello());
    }

}
```

The registered handler instances are not contextual intances, i.e. they're not managed by the CDI container (similarly as Java EE components). However, the dependency injection is supported.

#### How does it work?

The central point of integration is the `org.jboss.weld.vertx.web.WeldWebVerticle`. This Verticle extends `org.jboss.weld.vertx.WeldVerticle` and provides the `WeldWebVerticle.registerRoutes(Router)` method:

```java
 class MyApp {

     public static void main(String[] args) {
         final Vertx vertx = Vertx.vertx();
         final WeldWebVerticle weldVerticle = new WeldWebVerticle();

         vertx.deployVerticle(weldVerticle, result -> {

             if (result.succeeded()) {
                 // Configure the router after Weld bootstrap finished
                 Router router = Router.router(vertx);
                 router.route().handler(BodyHandler.create());
                 weldVerticle.registerRoutes(router);
                 vertx.createHttpServer().requestHandler(router::accept).listen(8080);
             }
         });
     }
 }
 ```
