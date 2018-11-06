# Embracing Async Await in WPF

Async/await is a feature of many modern programming languages that makes reasoning about asynchronous code way easier. In WPF this can be a pain, however, because of the 'viral' nature of async code and the non-thread-safe nature of UI code. "It's async all the way down"... until it isn't. 

The UI components in WPF or WinForms style applications are not thread safe, so all changes to the UI state must occur on the "UI Thread." In WPF threads can "dispatch" to the UI thread using a static reference to the running application (`App.Current.Dispatcher.Invoke`). This has 2 major issues, 
 1. You need a reference to a running application (sorry unit tests)
 2. The dispatched action cannot be async, meaning at some point you will still need to interact synchronously with asynchronous actions. 
 
The built in `.Result` and `.Wait()` members on `System.Task` will block until the result is available, but because of WPFs threading model these will almost always create deadlocks. `task.GetAwaiter().GetResult()` will also deadlock without calling `task.ConfigureAwaitable(false)`, which is a nuisance and a work around that is also prone to failing. 

My solution has been to provide configurable extension methods on `Task` for dispatching to *some dispatcher*, treated as any `Action<Action>`. App.Current.Dispatcher is an Action<Action>, so the extension methods can be configured (currently statically) at startup to dispatch to App.Current.Dispatcher, or to whatever other Action<Action> the developer needs them to. The extension methods allow for 'pipelining' asynchronous operations, dispatching to the UI thread, or unwrapping exceptions. 

## Usage

During application startup,
```
StaticDispatch.RegisterDispatcher(App.Current.Dispatcher.Invoke);
```

`.Confirm` takes callbacks for errored, canceled, or succesfull tasks,
```c#
async Task IoThingAsync() => await Task.Delay(500);

IoThingAsync()
    .Confirm(
        // will throw on the UI thread so exception does not get swallowed by the async goblins    
        errorCallback: e => throw e, 
        // will be called if the task completes succesfully
        successCallback: () => Console.WriteLine("Gottem!"),
        // will be called if the task is canceled
        cancelCallback: () => Console.WriteLine("Unlucky...")
    );
```

`.Then` takes an action to be dispatched when the task completes, or throws any errors it unwraps on the UI thread,
```c#
async Task<int> GetNumberAsync() => 1;

GetNumberAsync()
    .Then(i => {
        viewModel.UiProperty = i;
    });
```

Note that constant dispatching or context switching will greatly degrade the *actual* performance benefits provided by async/await in situations where the "async all the way down" mantra can be fully embraced. I emphasize 'actual' performance because to the applications users the *apparent* performance experienced may still be improved by removing cases where the UI locks up as expensive operations are forced on the UI thread. Using async/await and these extensions in WPF should be for the improved clarity and code quality - not necessarily for performance boosts.

## Feedback/Contributions

If you'd like to provide comments, suggestions, or code to `Kelson.Common.Async` I'd love to see issues opened on [this repository](https://github.com/KelsonBall/Kelson.Common.Async)

If you'd like to comment on this blog post or submit an edit, open an issue on [Kelson.Blog](https://github.com/KelsonBall/Kelson.Blog)

If you'd like to try this in your project, check out the nuget package `Kelson.Common.Async 0.1.1`