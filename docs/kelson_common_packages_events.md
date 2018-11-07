# Highlights from the new Kelson.Common packages

While working on a port and extension of a clients legacy financial services software I've developed a collection of reusable packages for enabling quick windows client development in WPF with a highly modular UI, reporting services, and asynchronous data access. While organizing and publishing these packages I've added a number of previous projects under the same `Kelson.Common` namespace.

The public versions of these packages should be considered "Work in Progress" until more complete documentation and guides are released.

If you are interested in providing feedback or contributions to any of these packages please contact me on the discord server linked in the footer of [my website](https://www.kelsonball.com) or open an issue on the related projects [repository](https://github.com/KelsonBall/Kelson.Common.Events).

# Kelson.Common.Events

After running in to some lifetime issues with another events and IoC container package I decided to flush out an event bus implementation I had been exploring for my own learning. The resulting `Kelson.Common.Events` package provides an interface for several decoupled eventing patterns, including the traditional Pub/Sub pattern, a Request/Response pattern, and a Provider pattern. 

The purpose of an event bus or aggregator is to allow different application components to communicate without requiring direct coupling of those components. The can enable more modular and flexible architectures, which in turn greatly simplifies the unit testing process. 

## Publish/Subscribe

The Pub/Sub eventing model is used for cases where application components want to "broadcast" events without requiring any knowledge about the events recipients. Similarly, components can subscribe to events without requiring any knowledge of the source of those events. 

Events are published and subscribed to based on their payload type. In this code sample an event is published using the type 'int', and then a similar event is published using a more descriptive wrapper type around an integer to specify exactly what the event means.

Pub/Sub code sample:

```c#
using Kelson.Common.Events;

// ... 

var events = new EventManager();

// no subscribers, nothing happens, and that is O.K.
events.Publish<int>(3);

// a subscription is registered
events.Subscribe<int>(i => Console.WriteLine(i));

events.Publish<int>(4);
> 4

// using a specific event type is important. 
// Not creating specific event types is similar to polluting a global namespace
public class ExampleEvent 
{ 
    public int Value { get; set; }
}

events.Subscribe<ExampleEvent>(e => Console.WriteLine(e.Value));

events.Publish(new ExampleEvent{ Value = 2 });
> 2
```

Calling the `IEventManager.Subscribe<T>` method will return an `ISubscription` object with an `Unsubscribe` method allowing channels to be explicitly unsubscribed from. If the desired behavior is instead for the subscription to be automatically dropped when an object goes out of scope you can use the `IEventManager.Listen<T>` method instead and hold a reference to the returned `ISubscription`. When that token is garbage collected the subscription will be automatically dropped by the event manager.

## Request / Response

Another useful eventing use-case is for requesting a result from some request handler. This is the eventing pattern used by HTTP for web traffic. In the case of the web a request expects exactly 1 response, meaning if no handler is available for a request then the request will fail. 

To maintain the event manager as a tool for decoupling application architectures the `IEventManager.Request<TRequest, TResponse>` method returns an enumerable of results, meaning no 'knowledge' is required of how many handlers are provided for a given request type. If there are no handlers the result will be an empty collection of responses. If there are many handlers available for a given request then the result will contain a response for each handler. It is to the subscriber to check if any results are returned and to enumerate the results.

```c#
var events = new EventManager();

public class ParameterRequest
{
    public string Key { get; set; }
}

events.Request<ParameterRequest, string>(new ParameterRequest { Key = "Text" })
> Enumerable<string> ( 0 ) [ ] 


events.Handle<ParameterRequest, string>(r =>
    {
        if (r.Key == "Name")
            return "Kelson";
        else if (r.Key == "Text")
            return "Wrote this post";
        else
            return null;
    });

events.Request<ParameterRequest, string>(new ParameterRequest { Key = "Text" })
> Enumerable<string> ( 1 ) [ "Wrote this post" ]
```

The `IEventManager.Request` method does return a true enumerable that won't be evaluated until invoked. For example, if `.First()` is called on the result enumerable then only the first handler will be invoked. Handlers should be considered unordered.

```c#

events.Handle<string, string>(s => 
    {
        Console.WriteLine($"Handler 1: {s}");
        return s;
    });

events.Handle<string, string>(s => 
    {
        Console.WriteLine($"Handler 2: {s}");
        return s;
    });

events.Request<string, string>("hello").First()
> Handler 1: hello
> "hello"
```

## Provide / Request

Provide/Request is similar to the Request/Response pattern but with a stricter interface. There is no request type passed to the handlers, instead handlers (called "providers" here) are invoked based on their return type, and each type is expected to have exactly 1 provider.

One use case of this pattern is to allow the event manager to be used as a flexible and less prescriptive alternative to a full IoC container. 

```c#
var events = new EventManager();

// provide a new Doodad on each request
events.Provide<IDoodad>(() => new Doodad());

// create a widget and return it for all IWidget requests
var myWidget = new Widget();
events.Provide<IWidget>(() => myWidget);

// request a widget and a doodad
var widget = events.Request<IWidget>();
var doodad = events.Request<IDoodad>();
```