# Using Configure Await in ASP.NET 
January 2018
__Syntax__
``` 
public ConfiguredTaskAwaitable ConfigureAwait(
	bool continueOnCapturedContext
)
Parameters
continueOnCapturedContext
   true to attempt to marshal the continuation back to the original context captured; otherwise, false.
```
The purpose of this document is to explain what configure await does in the web context and when to use it.   

## Context 
By default, when an incomplete Task is awaited, the Synchronization Context is captured. The type of context is dependent on technology you are working with. For this blog post we will be focusing on AspNetSynchronizationContext in ASP.NET.

When the task resumes, it first enters the captured context before executing the remainder of the code. In fact, AspNetSynchronizationContext guarantees your continuations will get the same HttpContext.Current even if your task continues on a different thread. 

 
## What does Configure Await do?
The parameter name continueOnCapturedContext hints at what it does. With configure await false, when an incomplete task is awaited the current context is not captured and is not available on when the executing the method. 

The following API controller shows where context is dropped.
```CSharp
 // GET api/values
public async Task<IEnumerable<string>> Get()
{
    //Warning bad code

    //Context is present here. So you will get data.
    var httpContext = HttpContext.Current;
    httpContext.Request;
    httpContext.User;
    httpContext.Response;
    httpContext.AllErrors;
    httpContext.Session;
    
    await DoSomethingAsync().ConfigureAwait(false); //Will drop Context here.
    //Context is null here. 
    httpContext = HttpContext.Current;
    return new List<string>();
}

public async Task DoSomethingAsync()
{
    await Task.Delay(1000);
}
```
 
The right time to use ConfigureAwait is when you know that you are not going to need the context and never in top level methods such as Controllers. 

This code shows when it maybe Ok to use configure await. 
```CSharp
// GET api/values
public async Task<IEnumerable<string>> Get()
{
    //Context is present here. So you will get data.
    var httpContext = HttpContext.Current;
    //The context will be captured here.
    await DoSomethingAsync();
    //Context is present here. Because it was capture when the task was awaited.
    
    httpContext = HttpContext.Current;
    return new List<string>();
}

public async Task DoSomethingAsync()
{
    //Context is present here.
    await DoSomethingElseAsync().ConfigureAwait(false); //Context for this function dropped here.
    //No Context here
    //HttpContext.Current is null here
}

public async Task DoSomethingElseAsync()
{
    //No Context here
    await Task.Delay(1000);
}
```

## Using ConfigureAwait
In most cases default behavior is the right way to go.  You should definitely not use ConfigureAwait when you have code after the await in the method that needs the context. For ASP.NET apps, this includes any code that uses HttpContext.Current or builds an ASP.NET response, including return statements in controller actions.

If you have a core library that’s potentially shared with desktop applications, consider using ConfigureAwait in the library code.

## What about Deadlocks 
Using ConfigureAwait(false) to avoid deadlocks is a dangerous practice. You would have to use ConfigureAwait(false) for every await in the transitive closure of all methods called by the blocking code, including all third- and second-party code. Using ConfigureAwait(false) to avoid deadlock is at best just a hack. 

## Further Reading/Resources
[Async/Await - Best Practices in Asynchronous Programming - By Stephen Cleary | March 2013](https://msdn.microsoft.com/en-us/magazine/jj991977.aspx)

[Parallel Programming with .NET by Stephen Toub | June 15, 2012 ](https://blogs.msdn.microsoft.com/pfxteam/2012/06/15/executioncontext-vs-synchronizationcontext/)

[Parallel Computing - It's All About the SynchronizationContext By Stephen Cleary | February 2011](https://msdn.microsoft.com/en-us/magazine/gg598924.aspx)

[Best practice to call ConfigureAwait for all server-side code](https://stackoverflow.com/questions/13489065/best-practice-to-call-configureawait-for-all-server-side-code)



