# Reentrancy

`Context.ReenterAfter` allows you to combine asynchronous operations with the actor-models single-threaded semantics.

Instead of `await`-ing a .NET `Task`, which would block the actor from processing more messages while the task is running.
You can instead allow the completion of the task to be scheduled back into the actors concurrency control.

Behind the covers, `Context.ReenterAfter(task, callback)` uses a task continuation and pass the callback as a message, back to the actor itself.

**Simplified pseudo implementation of ReenterAfter:**

```csharp
someTask.ContinueWith(t => context.Send(context.Self, new Continuation(callback, someTask)))
```

Imagine the following code:

```csharp
public async Task Receive(IContext context)
{
    if (context.Message is DoStuff)
    {
        var data = await MyDataAccessLayer.GetSomeData(...);

        //do something with the result
        //....
    }

    //Exit
}

```

This would execute as follows:

```mermaid
    graph TB
    receive(Receive message)
    start(GetSomeData starts)
    blocked(Task is executing<br>actor is blocked)
    class blocked red
    rest(do something with the result)
    exit((Exit))
    class exit gray

subgraph ac [Actor Concurrency]
    style ac fill:#00000030, stroke-dasharray: 5, stroke: #ffffff, stroke-width:2px
    receive --> start --> blocked ----> rest --> exit
end

```

While if we introduce `ReenterAfter`, like so:

```csharp
public async Task Receive(IContext context)
{
    if (context.Message is DoStuff)
    {
        var dataTask = MyDataAccessLayer.GetSomeData(...);
        context.ReenterAfter(dataTask, data => {
            //do something with the result
            //....
        });
    }

    //Exit
}

```

We instead get this execution flow:

```mermaid
    graph TB
    receive(Receive message)
    start(GetSomeData starts)
    await(Reenter into the actor thread)
    blocked(Task is executing)
    class blocked green
    rest(do something with the result)
    exit((Exit))
    class exit gray
    free1(Other messages can be processed here)
    class free1 comment


subgraph ac [Actor Concurrency]
    style ac fill:#00000030, stroke-dasharray: 5, stroke: #ffffff, stroke-width:2px
    direction TB
    receive --> start --> exit -->free1-->rest
end

start -.-> blocked -.-> await -.-> rest


```
