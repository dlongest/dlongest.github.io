---
layout: post
title: Introduction to Custom Specimen Builders in AutoFixture
---
##{{ page.title }}
_{{ page.date | date: "%-d %B %Y" }}_

It is no secret at work that I greatly admire Mark Seemann.  Anyone that comes to me and asks about software design I steer them right to his book "Dependency Injection in .NET".  In some respects, the title does it a bit of a disservice as at least half of it discusses some critical component design practices that apply regardless of whether one wants to use dependency injection.  Don't get me wrong, he covers dependency injection quite comprehensively, but there's a lot more to DI than just picking a container.  Beyond his book, Mark is an incredibly accessible guy in a couple respects.  One, he blogs pretty regularly and has a lot of incredible posts on design, I find myself going back to them again and again.  And two, he is the author of AutoFixture, which needs almost no introduction in the .NET world.  And it is this library that is the subject of this post. 

When I explain AutoFixture to other developers, I start by emphasizing its most straightforward use: when writing unit tests, it will provide you with test data with almost no setup, as if by magic.  Whether you needs integers or strings or collections or even complex objects, AutoFixture will provide them to you in a semi-random fashion.  Within the AutoFixture codebase, the components that handle creation of some value are termed specimen builders and they inherit `ISpecimenBuilder`.  A few common examples:  

* `Int32Generator` is the specimen builder that generates integers, beginning at 1 and incrementing by 1 for each subsequent integer that the fixture creates.   
* `StringGenerator` generates strings and the default randomization is to append a GUID to the name of the property or field
* `RandomRangedNumberGenerator` generates a random number within a given range, but doesn't repeat numbers in that range until all values have been used (shameless plug: I submitted this to AutoFixture)

And of course there are plenty more beyond that that do a variety of things.  Mark has an older but still valid post from 2010 on how to approach creating a custom specimen builder, but I think there's some nuances to it that aren't entirely clear to new users of AutoFixture.  I know when I introduced AutoFixture at work we used it in a sub-optimal manner because it wasn't always clear how to best customize it for certain situations and I want to save others some effort in the future since there isn't always time to dig into new libraries like one would hope.  This in no way takes away from how incredible AutoFixture is or how responsive Mark and others (Adam Chester and Nikos Baxevanis among them) are to questions about it on Stack Overflow or elsewhere, just my attempt to contribute some first-hand experience and perspective.

The `ISpecimenBuilder` interface has the following signature (taken directly from the AutoFixture codebase):
```csharp
	public interface ISpecimenBuilder
    {        
        object Create(object request, ISpecimenContext context);
    }
```

Here, `request` is *something* that needs to be created (and I'll come back to that in a minute).  The `context` can be used if necessary to generate other values from the fixture, but is not always needed.  For example, `Int32Generator` has no need of the `context` as it contains all the knowledge for satisfying its creation goals.  But suppose you're creating a specimen builder for a complex type and in the course of that, you want AutoFixture to provide some value (of any type)?  In that case you can use the `context` to obtain it (I'm intentionally omitting why you get `ISpecimenContext` and not `IFixture`).  

Request is ambiguously typed as `object` - what is it?  Typically, `request` is one of:

* A `Type`
* A `PropertyInfo`
* A `ParameterInfo`

And that brings us finally to the point of this post:  why on earth is that and what does it mean for us trying to create a custom specimen builder?

We'll start with the most obvious, Type.  At the most basic level, use of AutoFixture generally looks like:

```csharp
	var fixture = new Fixture();            
    var contact = fixture.Create<Contact>();
```

We create a Fixture, then we ask it for the type `Contact`, which is defined as:

```csharp
	 public class Contact
    {
        public Contact(string personId)
        {
            PersonId = personId;
        }

        public string PersonId { get; private set;  }
    }
```

AutoFixture now attempts to satisfy the request for a contact by running it through a variety of specimen builders in an implementation of the powerful chain of responsibility pattern.  The chain in this case is (among other things) a collection of specimen builders that individually assess whether they are capable of providing a value of type `Contact`.  The request to create it falls through each builder in the chain until one finally returns an actual `Contact` (each returns a `NoSpecimen` if it cannot handle the request).  But what does it mean to create a Contact?  

As shown, AutoFixture will find that Contact has one constructor and that this constructor takes a string.  So AutoFixture will automatically attempt to resolve creation of a string, then use that to create the `Contact` via its constructor.  This would be true regardless of how many dependencies the constructor had or their type.  And if there are multiple constructors, AutoFixture selects among them (by default, taking the one with the fewest parameters).  So from that view, AutoFixture creating an instance of an arbitrary complex type is simply the creation of any dependencies of that type until the entire graph is constructed. 

Now let's say for the sake of argument that PersonId, despite being typed as a string, is actually numeric.  We've already seen earlier that AutoFixture, by default, will make strings whose value are the property name and a GUID so we know that the default is not going to work for us here.  The first and easiest approach to this is:

```csharp
	var fixture = new Fixture();
    fixture.Register(() => "12345678");
    var contact = fixture.Create<Contact>();
```

This solves one problem (PersonId on the Contact will be "12345678") *except* this will cause AutoFixture to use "12345678" for every string it creates.  So we've overreached a bit here.  Of course we could use some method that generates a random integer and uses that instead of a hard-coded one, but again, it will use that random numeric string everywhere.  At some point, it might occur to us "What if we could customize the creation of Contact as a whole instead of at the string level?"  Enter a custom specimen builder.  

```csharp
	public class ContactGenerator : ISpecimenBuilder
    {
        public object Create(object request, ISpecimenContext context)
        {
            var type = request as Type;

            if (type == null)
                return new NoSpecimen(request);

            if (type != typeof(Contact))
                return new NoSpecimen(request);

            return new Contact("12345678");
        }
    }
```

So here, what is `request`?  Well remember, we're asking the Fixture to resolve `Contact` for us and our specimen builder's purpose is to create an entire `Contact` so `request` would be a Type.  But it's not just any arbitrary `Type`, our builder would only care about `Contact`.  The builder above follows a standard form:  attempt to cast `request` into what it's able to operate on, ensure that we did indeed get a request we can handle (returning `NoSpecimen` if we can't), and then create our desired instance.  In the above, we're hardcoding "12345678" as the parameter to the `Contact` constructor.  What if we want AutoFixture to make a random integer for us to use for that purpose?  Simple enough to use the `context` to resolve that for us:

```csharp
	return new Contact(context.Create<int>().ToString());
```

Now we're able to create a `Contact` with a properly numeric PersonId whose value is being provided by the fixture's normal integer rules.  The final step is to tell AutoFixture about the specimen builder:

```csharp
	fixture.Customizations.Add(new ContactGenerator());
```

Now you have a specimen builder capable of constructing a given type for you and perhaps that is sufficient for your needs.  But let's say `Contact` gets a bit more interesting:

```csharp
	public class Contact
	{
		private readonly string personId;
		private readonly int level;
		private readonly string firstName;
		private readonly string lastName;
	
		public Contact(string personId, string firstName, string lastName, int level)
		{
			/// elided
		}
		
		/// normal get; private set; property declarations	
	}
```

So here our contact is a PersonId (using the same numeric string rules as before), a level (an integer from 1 to 3), and first name and last name.  Let's assume that names are not particularly interesting, but there are valid reasons to use different levels in your testing, but again, they must be constrained within a specified range.  

We can let AutoFixture just provide us with random strings for the names, but we cannot simply ask it for an integer for level as we must constrain it to a certain range.  How can we support that?  Of course one way is to extend our `ContactGenerator` class to now include rules for constraining levels.  That might look like:

```csharp
	 return new Contact(context.Create<int>().ToString(),
                               context.Create<string>(), context.Create<string>(),
                               new Random().Next(1, 3));
```

This seems to work fine, but something about it feels a little off.  For one, we've got a lot of rules about our system and its valid inputs buried within this builder instead of having them be explicit.  Sometimes, this suggests the need to make a concept more explicit, possibly by introducing `PersonId` and `Level` as value objects that properly communicate our intent.  Let's say we introduce such types as shown below:

```csharp
	public class PersonId
    {
        public PersonId(string personId)
        {
            if (!personId.IsNumeric())
                throw new ArgumentException("personId");

            this.Id = personId;
        }

        public string Id { get; private set; }
    }

    public class ContactLevel
    {
        public ContactLevel(int level)
        {
            if (level <= 0 || level > 3)
                throw new ArgumentException("level");

            this.Level = level;
        }

        public int Level { get; private set; }
    }
```

That changes our `Contact` constructor to:

```csharp
	public Contact(PersonId personId, string firstName, string lastName, ContactLevel level)
```

Now asking AutoFixture to resolve `Contact` will cause it to resolve `PersonId`' and `ContactLevel`.  But aren't we back where we started?  Again, `PersonId` must take a numeric string and `ContactLevel` must take an integer between 1 and 3.  Should we extend `ContactGenerator` to include that logic?  In my opinion, definitely not.  `ContactGenerator` doesn't have any special rules around the `PersonId` and `ContactLevel` it can take, it just requires that it receive instances of each so imbuing it with the rules for creating a Contact don't really make sense.  
