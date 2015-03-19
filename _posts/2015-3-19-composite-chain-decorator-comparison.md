---
layout: post
title: Refactoring Comparison - Decorator vs Composite vs Chain of Responsibility
tags: design
description: A comparison of three powerful design patterns, decorator, composite, and chain of responsibility using a real-life example. 
---
In a <a href="2015/03/17/basics-decorator-pattern.html">prior post</a> I discussed in a bit of detail the Decorator pattern and how its used to add behavior to components, extending them in ways that weren't foreseen initially.  My plan had been to write a similar post on two other design patterns that I don't see used as much, but I've found to be incredibly useful in a lot of situations:  Composite and Chain of Responsibility.  However, sometimes an opportunity presents itself as one did today.  I've been teaching .NET development to an otherwise experienced database developer on my team and an item he picked up recently is tailor made for a comparison between these three patterns; I actually can't think of a better way to showcase them than against the same problem.  

Since we previously established how the Decorator works, I need to at least say a few quick words about the other patterns.  Taking Chain of Responsibility first, it's almost exactly what it sounds:  you take some number of components that implement a given interface and you configure them together (so the first calls second, second calls third, etc to the last one).  Each component looks at its inputs and decides if it should take some action and if it can, it does.  At that point depending on the implementation style, the component can either pass it to the next component in the chain or stop execution at that point.  That might sound a little vague, but it's because the chain is incredibly flexible and different situations call for different usage patterns.  The keys are:  each component has a next component it can call; its decision to call or not call the next component depends on exactly how the problem is decomposed across the components, either may make sense. 

A Composite, when you learn about it in an academic sense, is usually taught with a very poor example.  At a high level, a composite is about treating a group of components as if they're a single component.  For example, the classic example is a Windows form that itself contains a variety of controls.  If the form is moved or resized, each component may needs it Redraw method called to make the proper adjustment.  Instead of forcing the client component to know about and call Redraw on every individual piece, the form serves as a composite that takes the responsibility of calling Redraw on all its child components.  This example, while true, never proved that useful to me in understanding this pattern because it implies some kind of graph or tree relationship (forms to controls; some kind of tree structure) that doesn't per se have to exist to implement this.  I'm struggling to find the link at present, but Mark Seemann on his blog had an entry that finally clicked for me.  Suppose we have a collection of components and we'd like to execute them against some input, but we'd like to present them to the client as a single component? Enter the Composite.  So if we had `ICanDo`, we'd have some number of implementations of that interface plus one `CompositeCanDo`, which would take the collection of components and when called, execute the input on all (or selected) sub-components, and returning output as designed by the composite (application dependent).  Lot of description so let's get to an example.  

Here's a version of the code we have:

````c#

public class Registration
{
	public string PersonID { get; set; }
	public string FirstName { get; set; }
	public string LastName { get; set; }
	public string TestLocation { get; set; }
	public string ExamCode { get; set; }
	
	/// bunch more properties
}

public class ApplicationLogic
{
	public Registration GetRegistration(string personID, string examCode)
	{
		var repository = new RegistrationRepository();		
		
		try 
		{			
			return repository.GetByPersonIDAndCode(personID, examCode);		
		}
		catch (Exception ex ) { // do some stuff, ultimately rethrow }	
	}
	
	// Lot more unrelated methods
}

````

So first, yes the class in question really is named essentially `ApplicationLogic`, which is a poor name.  Also it has a ton of unrelated methods, basically each one exactly matches a method at the top-level WCF service that's using this class.  Second, yes, it does instantiate from scratch basically a repository instance (it's not called that though), then calls a method on it to retrieve a `Registration`, then returns it.  There's actually a separate repository mixed in there as well whose value I cannot immediately ascertain (because the names for all of them are not helpful; they include the suffix `DAO`).  

Anyway, here's the issue.  The client that's calling this service is showing a lot of data in a basic form to internal users.  It's actually showing the `TestLocation` twice, once sourced from this service (which gets it from the database via the repository) and once from calling a separate service.  The issue is the separate service is basically a persistence cache of person data since our legacy CRM is too slow to provide it to our enterprise site, but by virtue of that, it contains a flat view of the person.  However, the internal form needs to show registrations for any past exam code, but the cache service always only knows about the current one.   So what we'd like to do is insert a call into this service to also call the legacy CRM to fetch its test location, which we would include in our output (so it would return two test locations).  If you imagine just sticking the code into the existing code, might make it look like:

````c#

public Registration GetRegistration(string personID, string examCode)
	{
		var repository = new RegistrationRepository();		
		
		try 
		{			
			var registration = repository.GetByPersonIDAndCode(personID, examCode);
			
			/// Add the below two lines
			var otherSystemRegistration = new CrmServiceProxy().GetRegistration(personID, examCode);			
			registration = MergeRegistrations(registration, otherSystemRegistration);
			
			return registration;
		}
		catch (Exception ex ) { // do some stuff, ultimately rethrow }	
	}

````

Hopefully we've successfully outlined the problem facing us so we can get to the interesting part.  I'll now take the above code and demonstrate how we could refactor it into a Decorator, Composite, and Chain of Responsibility pattern.  I will also point out some considerations that might lead us to favor one over the other for this particular problem.  

Before we start, let's assume we've augmented our `Registration` class to include our new test location property on top of the existing properties that we'll call `CrmTestLocation`:

````c#

public class Registration
{
	public string PersonID { get; set; }
	public string FirstName { get; set; }
	public string LastName { get; set; }
	public string TestLocation { get; set; }
	public string ExamCode { get; set; }
	public string CrmTestLocation { get; set; }
	
	/// bunch more properties
}

````

<h3>Decorator</h3>

Let's arbitrarily start with the Decorator pattern since we covered it first.  As a reminder, the decorator starts with a base component and then decorates it with some additional behavior.  Since we're going to rely on some form of our existing repository logic to populate most of the fields and only use our new logic to add the CrmTestLocation.  Since both calls take a `PersonID` and `ExamCode`, we can establish this interface:

````c#

public interface IGetRegistration
{
	Registration Get(string personID, string examCode);
}

````

With this interface in hand, we know we'll need two implementations, one to hold the base logic and one that decorates that logic with the CRM service piece.  This lands us with these implementations:

````c#

public class GetRegistrationFromDb : IGetRegistration
{
	private readonly ReportRepository repository;
	
	public GetRegistrationFromDb(ReportRepository repository)
	{
		this.repository = repository;
	}

	public Registration Get(string personID, string examCode)
	{
		try 
		{			
			return repository.GetByPersonIDAndCode(personID, examCode);			
		}
		catch (Exception ex ) { // do some stuff, ultimately rethrow }	
	}
}

public class GetRegistrationFromCrm : IGetRegistration
{
	private readonly ICrmProxy proxy;
	private readonly IGetRegistration inner;
	
	public GetRegistrationFromCrm(ICrmProxy proxy, IGetRegistration inner)
	{
		this.proxy = proxy;
		this.inner = inner;
	}
	
	public Registration Get(string personID, string examCode)
	{
		try 
		{
			var registration = inner.GetRegistration(personID, examCode);
			
			CrmRegistration crmRegistration = this.proxy.ExpensiveWireCallToGetRegistration(personID, examCode);
			
			registration.CrmTestLocation = crmRegistration.TestLocationField;
			
			return registration;
			
		}
		catch (Exception ex) { // rethrow }
	}
}

````

First, we take our existing database-driven logic and simply put it into a component we'll name `GetRegistrationFromDb`.  The only other change we'll make is to inject the repository into the constructor instead of instantiating it.  Now we'll take our new logic and put it into a component named `GetRegistrationFromCrm`.  You'll see that it both implements `IGetRegistration` <strong>and</strong> takes as a constructor argument an implementation of `IGetRegistration` and that's the key.  This component starts by calling the inner component to get the initial, mostly-formed `Registration` instance. Next, it makes its call to fetch data from the CRM.  Finally, it updates the `CrmTestLocation` property on the `Registration` and returns it.  The CRM component decorates the enrichment of `CrmTestLocation` onto the primary database-driven component.  We're free to implement additional decorators on top of these if they make sense (if we want logging, if we have a need for more fields in the future) if necessary, the key here is that `GetRegistrationFromDb` is the base component and fills out the bulk of the fields.  If in the future we wanted to source most of the fields from the CRM, we would need to alter the implementation (or make a new one) that doesn't require an inner component to operate and does not selectively choose a single field to use from its output.  For that reason it's a valid question whether the database retrieval truly is a base functionality such that decorating our behavior is the appropriate paradigm to follow; in our case for this application that's true and so this would be appropriate, but if more fields were split between the two systems or if there were any collisions between fields, it would be less clear. 

<h3>Chain of Responsibility</h3>

Solving this problem with a chain of responsibility is not as different from a decorator as you'd think.  For the chain, we'll still have discrete components linked together, but the primary difference is essentially each component will be aware that there is a next one and its logic will have to include whether it should or should not call it.  We instantly have a bit of a conceptual problem though.  The chain implies that there is a single Registration that is being passed through each component in the chain and each component updates certain properties based on its logic.  In the decorator, our interface signature didn't have to pass the `Registration` around but now either we need to or we'll have to do something a little trickier.  Let's start with this interface:

````c#

public interface IGetRegistration
{
	Registration Get(Registration underConstruction, string personID, string examCode);
}

````

Here's now the weirdness with doing it this way.  When we want to start the chain, we have to call

````c#

	var registration = this.getRegistration.Get(new Registration(), personID, examCode);
	
````

Because each step in the chain needs access to a `Registration` instance since it needs to enrich it, it must be passed in.  For most of the components, they'll simply receive it from the prior component, but the first in the chain (which won't know it's first) has to be provided with it externally.  We *can* get around that though if we revise our view of the chain.  You would expect this chain to do something like the below that fetches the registration from the CRM service, merges its results into the `registration` parameter, then passes it plus the other parameters into the next component.  

````c#

var crmRegistration = CallCrmAndGetRegistration(personID, examCode);
MergeInto(crmRegistration, registration);

return this.next.Get(registration, personID, examCode);

````

But what if we reversed the order?

````c#

var registration = this.next.Get(personID, examCode);

var crmRegistration = CallCrmAndGetRegistration(personID, examCode);
MergeInto(crmRegistration, registration);

return registration;

````

Now instead of executing our component logic and then calling the next component, we call the next component first, *then* execute our logic, and finally return the registration directly from our logic.  You'll see in the pseudo-call to `Get` that we no longer have to provide a `Registration` as a parameter as now it propagates from the end of the chain forward through the return value of the method.  We have to be aware that doing it this way will cause every call to be placed onto the call stack until we reach the first component, at which point the stack will unwind back to the original caller.  This is similar to how recursion works.  Since it's unlikely that we'll have thousands (or more) individual components in our chain, it's unlikely we'll overrun the stack the way you could with recursion, but we need to at least be mindful. 

All things considered I like the back-to-front chain a lot better in this particular case so that's what we'll do.   Our interface is thus the same as the decorator and we end up with 3 components (one for database retrieval, one for CRM service call, and one that simply seeds the others with a default `Registration` instance).  Interestingly you'll see that our `GetRegistrationFromCrm` component doesn't actually have to change (although we do modify the member `inner` to be called `next`).  At the bottom you'll see the code to register all the components together.  The key in these implementations is to register them last to first so we start with `GetRegistrationFromCrm`, provide it with `GetRegistrationFromDb`, and provide it with `SeedGetRegistrationChain`. 

````c#

public interface IGetRegistration
{
	Registration Get(string personID, string examCode);
}

public class GetRegistrationFromDb : IGetRegistration
{
	private readonly ReportRepository repository;
	private readonly IGetRegistration next;
	
	public GetRegistrationFromDb(ReportRepository repository, IGetRegistration next)
	{
		this.repository = repository;
	}

	public Registration Get(string personID, string examCode)
	{
		try 
		{			
			return repository.GetByPersonIDAndCode(personID, examCode);			
		}
		catch (Exception ex ) { // do some stuff, ultimately rethrow }	
	}
}

public class GetRegistrationFromCrm : IGetRegistration
{
	private readonly ICrmProxy proxy;
	private readonly IGetRegistration next;
	
	public GetRegistrationFromCrm(ICrmProxy proxy, IGetRegistration next)
	{
		this.proxy = proxy;
		this.next = next;
	}
	
	public Registration Get(string personID, string examCode)
	{
		try 
		{
			var registration = this.next.GetRegistration(personID, examCode);
			
			CrmRegistration crmRegistration = this.proxy.ExpensiveWireCallToGetRegistration(personID, examCode);
			
			registration.CrmTestLocation = crmRegistration.TestLocationField;
			
			return registration;			
		}
		catch (Exception ex) { // rethrow }
	}
}

public class SeedGetRegistrationChain : IGetRegistration
{	
	public Registration GetRegistration(string personID, string examCode)
	{
		return new Registration();
	}
}


var chain = new GetRegistrationFromCrm(
				new CrmProxy(), new GetRegistrationFromDb(
									new Repository(), new SeedGetRegistrationChain()));


````

<h3>Composite</h3>

One thing the decorator and chain of responsiblity have in common is at least some of the components are aware of the existing of the others and in fact, that is a critical part of the overall logic of the pattern.  In a Composite, that's not true as you will see: only the composite itself has knowledge of all the components and how to compose their results together.  We can actually use our same interface as before and roughly the same implementations as before (some slight tweaks). 

````c#

public interface IGetRegistration
{
	Registration Get(string personID, string examCode);
}

public class GetRegistrationFromDb : IGetRegistration
{
	private readonly ReportRepository repository;	
	
	public GetRegistrationFromDb(ReportRepository repository)
	{
		this.repository = repository;
	}

	public Registration Get(string personID, string examCode)
	{
		try 
		{			
			return repository.GetByPersonIDAndCode(personID, examCode);			
		}
		catch (Exception ex ) { // do some stuff, ultimately rethrow }	
	}
}

public class GetRegistrationFromCrm : IGetRegistration
{
	private readonly ICrmProxy proxy;
		
	public GetRegistrationFromCrm(ICrmProxy proxy)
	{
		this.proxy = proxy;		
	}
	
	public Registration Get(string personID, string examCode)
	{
		try 
		{						
			CrmRegistration crmRegistration = this.proxy.ExpensiveWireCallToGetRegistration(personID, examCode);
			
			return new Registration 
						{
							CrmTestLocation = crmRegistration.TestLocationField
						};
		}
		catch (Exception ex) { // rethrow }
	}
}

public class CompositeGetRegistration : IGetRegistration
{
	private readonly IList<IGetRegistration> getRegistrations;
	
	public CompositeGetRegistration(params IGetRegistration[] getRegistrations)
	{
		this.getRegistrations = getRegistrations.ToList();
	}
	
	public Registration Get(string personID, string examCode)
	{
		var registrations = getRegistrations.Select(personID, examCode).ToList();
		
		return MergeRegistrationsIntoOne(registrations);
	}
}

````

The above code shows off a Composite, but it's not a great fit in this case because there's one hard problem that isn't addressed:  once you get back the `Registration` instances from each component, how do you know which one is the source of which values?  The best you could do automatically is follow a rule that you will use the first non-null value for each property in the order the components exist in the composite.  This would work okay for our use-case, but it falls apart if there are collisions (multiple components with different values for the same properties and sometimes you need one to win and sometimes the other).  Remember that the composite has no special knowledge of what components it contains so unless we could specify some default property setting logic like we just discussed, we'd have to add an additional layer of metadata for mapping the values from the component.  You could argue this problem exists in other implementations too, but the difference is the component has an opportunity to decide for itself if it should or should not set a given value.  In the composite, the component makes that decision, but the logic of how to piece them together is separate and really complicates matters here.  

<h3>Which of these approaches is best?</h3>

First when we say best, we mean only for this particular problem.  Hopefully you've seen that they all work pretty differently and so they each shine in their own way.  I would say a composite is not a particularly great choice for this problem.  We don't have the problem yet, but it's easy to envision running into trouble when more components are added and how we would map their output to the final, consolidated output. And unfortunately the composite has no way to communicate that at present.  Beyond that, either the decorator or the chain seem to fit this problem so choice between them is no better than preference at this point given the requirements we have. 

This was pretty lengthy, but hope you enjoyed this refactoring exercise for the purpose of showing off the decorator, composite, and chain of responsibility.  If I come up with other different examples in the future I hope to do this again and further show how these techniques can be leveraged in a lot of creative ways.  



