---
layout: post
title: Choose Between Components At Runtime In Castle Windsor With Handler Selector
tags: design
description: In Castle Windsor, use a custom IHandlerSelector to choose between components at runtime in cases when you need to perform some kind of conditional component resolution, but both are eligible to be used in different cases. 
---
There are a lot of extensibility points in Castle Windsor, but one common challenge is to find the appropriate one for a given job.  I ran into such a case recently so it seemed worth sharing.  Imagine we have  WCF service and its set up as below where `IMainService` is our service contract whose implementation is `MainService` and it takes as a constructor parameter and implementation of `IDependencyService`.  Early on in our application's life we only have a single implementation of `IDependencyService` called `DefaultDependencyService`, but now we need to add a second version, `DependencyServiceVersion2`, and at runtime decide which implementation should be used.  

````c#
[ServiceContract]
public interface IMainService
{
    [OperationContract]
    string GetData(int value);
}

public class MainService : IMainService
{
	private readonly IDependencyService service;
	
	public MainService(IDependencyService service)
	{
		this.service = service;
	}
		
    public string GetData(int value)
	{
		return this.service.DoDependentAction(value);
	}
	
}

public interface IDependencyService
{
    string DoDependentAction(int i);
}

public class DefaultDependencyService : IDependencyService
{
    public string DoDependentAction(int i)
    {
		return "fixed text";
    }        
}

public class DependencyServiceVersion2 : IDependencyService
{
    public string DoDependentAction(int i)
    {
        return i.ToString();
    }
}

````

I'll constrain the problem a bit and say that the selection of `IDependencyService` implementation is based on a value in the `HttpContext` (let's say it looks for a specific header called `Version`).  The challenge is, in Windsor we can register both implementations, but how exactly can we instruct it when it should use one versus the other?  The point at which the registrations are performed (when the service is first loaded) isn't appropriate because `HttpContext` is per-request.  

Of course one choice is something like this:

````c#

public class ServiceSelectingDependencyService : IDependencyService
{
	private readonly IDependencyService ifTrue;
	private readonly IDependencyService ifFalse;

	public ServiceSelectingDependencyService(IDependencyService serviceIfTrue, IDependencyService serviceIfFalse)
	{
		this.ifTrue = serviceIfTrue;
		this.ifFalse = serviceIfFalse;
	}
	
	public string DoDependentAction(int i)
	{
		return SelectService().DoDependentAction(i);
	}
	
	protected virtual IDependencyService SelectService()
	{
		var request = HttpContext.Current.Request;

        var headerKeys = request.Headers.AllKeys;
						
        return (headerKeys.Any(a => a.Equals("Version") && request.Headers[a].Equals("v2"))) ? this.ifTrue : this.ifFalse;			
	}
}

````

That works around the problem by using another implementation of `IDependencyService` to choose between the two based on the value of the HttpContext.  This is minimally invasive to the real components and only requires us to ensure the registrations in Windsor are such that it resolves `IDependencyService` to `ServiceSelectingDependencyService` and that it takes the appropriate service implementation.  One way to do such a registration is:

````c#

container.Register(Component.For<IDependencyService>()
                            .ImplementedBy<ServiceSelectingDependencyService>()
                            .LifeStyle.Transient.Named("ServiceSelectingDependencyService")
                            .DependsOn(ServiceOverride.ForKey("serviceIfFalse").Eq("DefaultDependencyService"))
                            .DependsOn(ServiceOverride.ForKey("serviceIfTrue").Eq("DependencyServiceVersion2")),
                   Component.For<DefaultDependencyService>()
                            .Named("DefaultDependencyService")
                            .LifeStyle.Transient,
                   Component.For<DependencyServiceVersion2>()                                            
                            .Named("DependencyServiceVersion2")
                            .LifeStyle.Transient);
							
````

This would work okay, but it feels like there should be a way in Windsor to accomplish this that doesn't push the job into an application component.  Enter `IHandlerSelector`!   We start by writing a class that implements `IHandlerSelector`, which defines two methods.  The first, `HasOpinionAbout`, takes a possibly-null key and a service type and the selector should return true if it is able to select a handler based on those parameters, false otherwise.  Windsor guarantees it will call `HasOpinionAbout` to determine if the call should proceed to the next method in the signature, `SelectHandler`, which should choose among the provided `Handlers` and return the appropriate one to be used.  A `Handler` in this case is a component in Windsor that participates in instance construction.  There are some fine-grained rules around how Windsor creates components and in what order it uses certain pieces of the kernel or container so I encourage you to <a href="http://docs.castleproject.org/Windsor.Handler-Selectors.ashx">read the Windsor docs</a> for more details, but suffice it to say if a `HandlerSelector` isn't being called for you as you expect, you probably have customized Windsor in some other way and are probably well beyond me in your knowledge of it.  

Below is the implementation of a handler selector that we can use to select between instances of `IDependencyService`, the same basic logic we had put into our application component.  The only `Type` that we care about selecting a `Handler` for is `IDependencyService` and that is the sole basis for the logic in `HasOpinionAbout`.  To select a handler, our custom logic is we look for the value of a header called Version in the HttpRequest and select the right implementation from that.  One thing that's implicit here is we rely on the component being registered under a name as that is the way we select the handler from the collection.  Most of the time this type of implicit knowledge is bad (and we could control it a bit here by making the name a constant somewhere at the application root level), but at this level it may be acceptable.   Also shown are the updated Windsor registration and the needed line to add our `HandlerSelector` into Windsor. 

````c#

public class SelectDependencyServiceHandlerSelector : IHandlerSelector
{
    public bool HasOpinionAbout(string key, Type service)
    {
        return service == typeof(IDependencyService);
    }

    public IHandler SelectHandler(string key, Type service, IHandler[] handlers)
    {
        var request = HttpContext.Current.Request;

        var headerKeys = request.Headers.AllKeys;

        if (headerKeys.Any(a => a.Equals("Version") && request.Headers[a].Equals("v2")))
            return handlers.First(a => a.ComponentModel.Name.Equals("DependencyServiceVersion2"));
        else
            return handlers.First(a => a.ComponentModel.Name.Equals("DefaultDependencyService"));
    }
}  

container.Register(Component.For<IDependencyService>()
                                 .ImplementedBy<DefaultDependencyService>()
                                 .Named("DefaultDependencyService")
                                 .LifeStyle.Transient);

container.Register(Component.For<IDependencyService>()
                                 .ImplementedBy<DependencyServiceVersion2>()
                                 .Named("DependencyServiceVersion2")
                                 .LifeStyle.Transient);                       	

container.Kernel.AddHandlerSelector(new SelectDependencyServiceHandlerSelector());			
					 
````

I hope you've enjoyed this brief look at how to use Windsor for conditional component resolution.  There are a lot more extensibility points in Windsor so if you feel like the container must have some capability and you can't just find it, see <a href="http://docs.castleproject.org/Windsor.MainPage.ashx">the Windsor docs</a> for more information or shoot me an email and I will see what I can do. 
