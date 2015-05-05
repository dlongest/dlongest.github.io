---
layout: post
title: Registering Decorators, Composites, and Chains in Castle Windsor
tags: design, castle windsor
description: If you've been avoiding these powerful patterns because of the difficulty in registering them in Windsor, fear no more. 
---
As I show people Castle Windsor and how to unlock its power to write more loosely coupled components, most quickly grasp the idea of registering a single component type against a service.  That's certainly a necessary first step, but what I often see is once developers start using Windsor, the code-base explodes with violations of the Reused Abstractions Principle or the code base still contains the same tightly coupled logic that it always did.  I've discussed in previous posts how to identify cases where a decorator, composite, or chain of responsibility can make a dramatic difference on a codebase, but I often see developers run into trouble registering components for these patterns in Windsor.  And in fact I've had some developers tell me they abandoned that design simply because they couldn't get it registered in Windsor.  I found that wholly shocking that a tool we're using that's capable of loosening our coupling is contributing to making it worse.  To rectify that, I'm going to demonstrate in this post some ways to compose these patterns in Windsor.  Definitely read to the bottom though as ultimately there is one super-easy way to do it.

<h2>Chain of Responsibility</h2>

In a chain of responsibility, we have some number of components that all implement the same interface and each one is connected to one (or more) of the others.  Clients call the first component in the chain and based on the logic of the overall chain sub-system, the component can take some action and/or forward it to the next component.  Below is a basic set of components for a chain that we'll use.  In our case, clients would be handed an instance of FirstInChain, which would call SecondInChain, which would call LastInChain.  

```c#

  public interface IChain
    {
        string Get();
    }

    public class FirstInChain : IChain
    {
        public FirstInChain(IChain next)
        {
            this.Next = next;
        }

        public string Get()
        {
            return this.Next.Get();
        }

        public IChain Next { get; private set; }
    }

    public class SecondInChain : IChain
    {
        public SecondInChain(IChain next)
        {
            this.Next = next;
        }

        public string Get()
        {
            return this.Next.Get();
        }

        public IChain Next { get; private set; }
    }

    public class LastInChain : IChain
    {
        public LastInChain()
        {        
        }

        public string Get()
        {
            return "I'm done";
        }        
    }
	
```

The most important rule to understand for these registrations is, by default, Windsor will use the first registration for a type to determine how the object graph is composed.  Thus if we want to be given an instance of `FirstInChain`, it needs to be registered first;

````c#

	container.Register(Component.For<IChain>().ImplementedBy<FirstInChain>());

````

So now, if we call `container.Resolve<IChain>()`, we'll get an instance of `FirstInChain`, but since Windsor doesn't have way know what `FirstInChain` should be given (and it won't give it an instance of itself), our chain is not properly composed.  So how can we instruct Windsor about our dependencies from First->Second->Last?  Well one way is through naming each component and specifying them as dependencies.  Here's a unit test that will showcase this.  I have scrambled up the order of Second and Last to showcase that Windsor identifies the 
components using the specified dependency relationships. 

````c#

  [Fact]
        public void Register_ChainOfResponsiblity_UsingDependencyOnComponent_ForNamedComponents()
        {
            var container = new WindsorContainer();

            container.Register(Component.For<IChain>().ImplementedBy<FirstInChain>().Named("first")
                                        .DependsOn(Dependency.OnComponent("next", "second")));

            container.Register(Component.For<IChain>().ImplementedBy<LastInChain>().Named("last"));			
			
            container.Register(Component.For<IChain>().ImplementedBy<SecondInChain>().Named("second")
                                        .DependsOn(Dependency.OnComponent("next", "last")));
                        
            var component = container.Resolve<IChain>();

            var first = Assert.IsType<FirstInChain>(component);
            var second = Assert.IsType<SecondInChain>(first.Next);
            var last = Assert.IsType<LastInChain>(second.Next);            
        }
		
````

The above is typically the first step most people make when trying to register these components.  This will work fine, but it just feels a little ugly.  How else could we do it?  Here's the amazing part of Windsor:  it will compose the graph correctly, automatically, if you just order the components the way you want them to be resolved.  

````c#

	[Fact]
    public void AutoWire_ChainOfResponsiblity_BasedOnRegistrationOrder()
    {
        var container = new WindsorContainer();

        container.Register(Component.For<IChain>().ImplementedBy<FirstInChain>(),                               
                           Component.For<IChain>().ImplementedBy<SecondInChain>(),
                           Component.For<IChain>().ImplementedBy<LastInChain>());
                            
        var component = container.Resolve<IChain>();

        var first = Assert.IsType<FirstInChain>(component);
        var second = Assert.IsType<SecondInChain>(first.Next);
        var last = Assert.IsType<LastInChain>(second.Next);
    }
		
````

The fact that Windsor will auto-wire this escaped me for quite a long time, but this is what makes Windsor so incredible: it just works.  

The same is also true for the other design patterns as well.

<h2>Decorator</h2>


A decorator is a pattern where one component of an interface type relies on another instance of that component to do its work.  For example, we may have a class that fetches records from a database and another that caches those records.  The caching component can "decorate" its logic on top of the other component.  Below is a sample set of components we'll work with.  We want SecondDecorator to be on top and contain an instance of FirstDecorator, which will contain an instance of BaseDecoratableService.  

````c#

 public interface IDecoratableService
    {
        string Do();
    }

    public class BaseDecoratableService : IDecoratableService
    {
        public string Do()
        {
            return string.Empty;
        }
    }

    public class FirstDecorator : IDecoratableService
    {
        private readonly IDecoratableService inner;

        public FirstDecorator(IDecoratableService inner)
        {
            this.inner = inner;
        }

        public string Do()
        {
            return string.Concat(this.inner.Do(), "#FIRST#");
        }

        public IDecoratableService Inner { get { return this.inner; } }
    }

    public class SecondDecorator : IDecoratableService
    {
        private readonly IDecoratableService inner;

        public SecondDecorator(IDecoratableService inner)
        {
            this.inner = inner;
        }

        public string Do()
        {
            return string.Concat(this.inner.Do(), "@@SECOND@@");
        }

        public IDecoratableService Inner { get { return this.inner; } }
    }

````

The registrations for Windsor to auto-wire this are almost identical to the Chain:

````c#

    [Fact]
    public void AutoWire_Decorator()
    {
        var container = new WindsorContainer();

        container.Register(Component.For<IDecoratableService>().ImplementedBy<SecondDecorator>(),
                           Component.For<IDecoratableService>().ImplementedBy<FirstDecorator>(),
                           Component.For<IDecoratableService>().ImplementedBy<BaseDecoratableService>());       

        var service = container.Resolve<IDecoratableService>();

        var secondDecorator = Assert.IsType<SecondDecorator>(service);
        var firstDecorator = Assert.IsType<FirstDecorator>(secondDecorator.Inner);
		var baseService = Assert.IsType<BaseDecoratableService>(firstDecorator.Inner);
	}
	
````
           
<h2>Composite</h2>

A composite is a pattern where one component contains some number of instances of the same interface and depending on the logic, the composite may selectively call its contained components or call all of them.  The complication for Windsor is that it needs to supply a collection of instances and not just a single instance.  This requires us to add Windsor's CollectionResolver (if it hasn't been added previously), but otherwise is identical to the Chain and Decorator:

````c#

	public interface IComposableService
    {
        void Do(string s);
    }

    public class CompositeComposableService : IComposableService
    {
        public CompositeComposableService(params IComposableService[] services)
        {
            if (services == null)
                throw new ArgumentNullException("services");

            this.Services = services.ToList();
        }

        public void Do(string s)
        {
            this.Services.ToList().ForEach(a => a.Do(s));
        }

        public IList<IComposableService> Services { get; private set; }
    }

    public class FirstComposableService : IComposableService
    {
        public void Do(string s)
        {            
        }
    }

    public class SecondComposableService : IComposableService
    {
        public void Do(string s)
        {
        }
    }

	[Fact]
    public void AutoWire_Composite()
    {
        var container = new WindsorContainer();
        /// Must add the CollectionResolver or it will not work.
        container.Kernel.Resolver.AddSubResolver(new CollectionResolver(container.Kernel));

        container.Register(Component.For<IComposableService>().ImplementedBy<CompositeComposableService>(),
                           Component.For<IComposableService>().ImplementedBy<FirstComposableService>(),
                           Component.For<IComposableService>().ImplementedBy<SecondComposableService>());

        var service = container.Resolve<IComposableService>();

        var composite = Assert.IsType<CompositeComposableService>(service);

        Assert.Equal(2, composite.Services.Count());
        Assert.True(composite.Services.Any(a => a.GetType() == typeof(FirstComposableService)));
        Assert.True(composite.Services.Any(a => a.GetType() == typeof(SecondComposableService)));
    }
	
````

<h2>Wrap Up</h2>

So there you have it: Windsor's auto-wiring will automatically compose these 3 powerful patterns for you.  In fact it takes more work to *not* just let Windsor handle this.  Now yes, if you have some specialized resolution needs (such as you don't always want every component to be put into the Composite or something like that), then you'll have to do something extra, but otherwise, no longer any need to fear these patterns.  In a future post, I'll show how you can control conditional resolutions in Windsor as well for some more advanced scenarios. 




