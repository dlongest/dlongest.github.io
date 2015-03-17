---
layout: post
title: Basics of the Decorator Pattern
tags: design
description: How and when to use the Decorator design pattern
---
At this point the classic Gang of Four design patterns book is over 20 years old and it's still frequently cited as developer's favorite book on design.  And it's truly incredible the staying power it's had considering how the software industry loves to throw out old things and bring in new ones.  In most university classes when you learn programming (specifically object-oriented, which is the default paradigm taught now), the instructor typically spends quite a bit of time going over how inheritance works and giving students toy problems to demonstrate how it works (so the old "A Person is an Employee", that kind of thing).  But in the real world, inheritance seems to cause more problems than it's worth hence the GoF wisdom:  Prefer composition to inheritance.  That is a very powerful statement and I've only recently begun grasping its true nature.  The decorator is a great example of a pattern that always felt more academic to me until I finally had an a-ha moment about a year back.  The goal of this post is to show a real-world example where a decorator can be applied to improve the design and in a future post we will see how Windsor can make it even easier for us. 

To start, what is the decorator pattern?  At the core, the pattern allows us to add behavior to a given component without changing the operation of the original component.  We say that the new functionality is "decorated" onto the prior component.  Okay, that sounds great, but in what kind of situation can we detect a need for it?  Suppose we have the below `IStateService` that defines an interface for getting a list of states (U.S. states).  We also have a `DbStateService`, which looks up the states up from our database context.  

````c#

public class State
{
	public string Name { get; set; }
	public string Abbreviation { get; set; }
}

public interface IStateService
{
	IEnumerable<State> GetStates();
}

public class DbStateService : IStateService
{
	private readonly StateContext context;  // Some database context, such as EF
	
	public DbStateService(StateContext context)
	{
		this.context = context;
	}

	public IEnumerable<State> GetStates()
	{
		return this.context.States.OrderBy(a => a.Abbreviation);
	}
}

````

This seems straightforward enough: each call to `DbStateService` results in some query to the database to fetch the list of states and then return it.  Now suppose we decide that we should implement some caching of the states (since they don't change).  Here's an unfortunately common implementation I'm seeing all over our codebases:

````c#

public interface ICustomCache
{	
	void Save(string key, object value);
	bool Contains(string key);
	object GetValue(string key);
}

public class DbStateService : IStateService
{
	private readonly StateContext context;  // Some database context, such as EF
	private readonly ICustomCache cache;
	
	public DbStateService(StateContext context, ICustomCache cache, bool cacheActive = true)
	{
		this.context = context;
		this.cache = cache;
		this.CacheActive = cacheActive;
	}

	public IEnumerable<State> GetStates()
	{
		if (this.CacheActive && this.cache.Contains("states"))
			return (IEnumerable<State>)this.cache.GetValue("states");
	
		var states = this.context.States.OrderBy(a => a.Abbreviation);
		
		if (this.CacheActive)
			this.cache.Save("states", states);
				
		return states;
	}
	
	public bool CacheActive { get; set; }
}

````

The original `DbStateService` implementation has been augmented to support this concept of a `ICustomCache` (which is a stripped down wrapper around an in-process cache or whatever) as well as a flag to indicate whether the cache is or is not.  Here's what's not good about this approach.  First, we can refactor GetStates anyway we like, but `DbStateService` is now doing two things:  it is looking up states from the database *and* it's managing the states being saved to or used from the cache.  In my purview, that's two separate responsibilities and so we should split them out into two components: one that looks up items in the database, one that manages the cache as necessary.  In addition we have tightly coupled our use of caching around this database implementation.  If in the future wanted to reuse this caching logic but source the states from somewhere else, our caching cannot be reused as-is so our level of changes are much higher. 

It is here that the Decorator comes into play.  First, we'll go back to the original version of `DbStateService` that we had initially.  Next, our decorator will look like this:

````c#

public class CachingStateService : IStateService
{
	private readonly IStateService service;
	private readonly ICustomCache cache;
	private readonly string cacheKey = "states";
	private readonly object synch = new object();
	
	public CachingStateService(IStateService service, ICustomCache cache)
	{
		this.service = service;
		this.cache = cache;
	}
	
	public IEnumerable<State> GetStates()
	{
		if (this.cache.Contains(this.cacheKey))
			return (IEnumerable<State>)this.cache.Get(this.cacheKey);
	
		var states = this.service.GetStates();	
		
		lock (this.synch)
		{		
			if (!this.cache.Contains(this.cacheKey))		
				this.cache.Save(this.cacheKey, states);				
		}
		
		return states;
	}
}

````

We've now defined a second implementation of `IStateService` called `CachingStateService` that takes as a constructor argument some `IStateService` implementation and some `ICustomCache` implementation and determines what to return based on the list being in the cache or not (which it also updates).  The key points to make here about the decorator in general and `CachingStateService` specifically:

<ul>
<li>Both implements the interface and takes as a constructor argument an implementation; it delegates calls to its injected implementation when it can't satisfy the call itself</li>
<li>Relies on `IStateService` for its inner implementation and *not*  a concrete one (such as `DbStateService`); all `CachingStateService` requires is that some inner implementation can satisfy the contract to provide states when asked, it doesn't care where or how it gets them</li>
<li>We can support any number of layers of decorators by following this same pattern: `SecuredStateService` that somehow verifies the security level of the requesting user or `LoggingStateService` that adds some tracing or any permutation of behaviors we'd like</li>
<li>The `IStateService` abstraction is stronger because we now have two implementations of it, a much better adherence to the `Reused Abstractions Principle`
</ul>

The typical smell for when a decorator could be pulled into being are when a component is doing some core function *plus* it's doing some additional, semi-related job on top of that.  If I see that type of setup, I try to decide if we can push the core functionality into one component and our additional functions into a decorator.  The nice thing about a `decorator` is you can apply it pretty gently.  In our case when `DbStateService` was doing both DB retrievals and caching, it's not overly burdensome to break into two components as it's essentially just shifting the logic among two components.  Some unit tests will of course be affected and have to change, but in truth our tests should have indicated we had a design issue already (since at least as developed, all the interesting logic is related to caching, not retrieval).  

In a future post we'll look at how to configure decorators in Castle Windsor as well as the interceptor construct that Windsor provides that we can use to decorate across types.  
