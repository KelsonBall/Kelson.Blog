# Highlights from the new Kelson.Common packages

While working on a port and extension of a clients legacy financial services software I've developed a collection of reusable packages for enabling quick windows client development in WPF with a highly modular UI, reporting services, and asynchronous data access. While organizing and publishing these packages I've added a number of previous projects under the same `Kelson.Common` namespace.

The public versions of these packages should be considered "Work in Progress" until more complete documentation and guides are released.

If you are interested in providing feedback or contributions to any of these packages please contact me on the discord server linked in the footer of [my website](https://www.kelsonball.com) or open an issue on the related projects [repository](https://github.com/KelsonBall/Kelson.Common.Async).

# Kelson.Common.Async

## 1. 'Then'

The `Async` package contains two gems. The one I use most frequently is the `.Then` extension method. When using C#'s wonderful `async await` syntax in WPF applications it is very easy to deadlock when trying to wait for tasks to complete so that the results can be used to update the UI. The two solutions are to use `.ConfigureAwait` on every task (theoretically.. this didn't work well for me when I tried it), or to use `App.Current.Dispatcher.Invoke` which is a long and cringy reference to non configurable static state. 

My solution was to add a configurable extension method on `Task` and `Task<T>` that could invoke an action on a configured dispatcher. During application startup I can now configure `.Then` to dispatch to `App.Current.Dispatcher.Invoke` and in tests I can dispatch to an action specified in a TestFixture type. 

Updating the UI after an asynchornous IO operation is now a delight, looking something like this
```cs
client.GetThingAsync().Then(thing => UiProperty = thing.Value);
```

## 2. The smol actor

The second gem in the Async package is the cutest `Actor` implementation I can imagine. 

... it's a task. 

That's it. 

An "actor" is a paraell programming pattern where a single thread is given ownership of a mutable resource or asset. Since all operations on that mutable resource must occur within that single thread concurrent access is no longer possible. With my minimalist implementation it's possible to pass references to the mutable resource out of the actors context... so don't do that 😅

Here is the entire code base for my Actor<T> type

```cs
public class Actor<T> where T : class
{
    private readonly Task<T> asset;

    public Actor(T value) => asset = Task.Run(() => value);    

    public async Task Do(Action<T> action) => await asset.ContinueWith(t => action(t.Result));    

    public async Task<V> Do<V>(Func<T, V> action) => await asset.ContinueWith(t => action(t.Result));
}
```

It probably won't be winning any distributed programming or performance awards, but it *does* provide one-at-a-time access to non concurrent datastructures. 

Here's a minimalist example 

```cs
private Actor<HashMap<string>> namesActor = new Actor(new HashMap<string>());

async Task AddName(string name) => await namesActor.Do(names => names.Add(name));
    
async Task<bool> HasName(string name) => await namesActor.Get(names => names.Contains(name));

```

or

```cs
var mapActor = new Actor(new Dictionary<string, string>());

foreach (var value in some_list_of_values)
    someRepository.GetAsync(value)
        .ContinueWith(async t => 
            await mapActor.Do(map => 
                map.Add(t.Result.Key, t.Result.Value)));

```

Remember to head over to [my website](https://www.kelsonball.com) to see what other projects I'm working on or to get in touch. 