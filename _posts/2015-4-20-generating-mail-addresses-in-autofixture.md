---
layout: post
title: Generating Mail Addresses in AutoFixture
tags: autofixture
description: The dangers of validating email addresses and how AutoFixture produces mail addresses for you.
---
I'm wrapping up stage 1 of some modifications to AutoFixture that slightly modify new logic it has for generating a `MailAddress`.  In case you're not familiar, `MailAddress` is the standard framework type for representing email addresses.  A prior pull request to AutoFixture added a new builder that produces these mail addresses, and essentially it did so this way:

````c#

	var name = Guid.NewGuid().ToString();
	var domain = this.fictitiousDomains[(uint)name.GetHashCode() %3];
	
	var email = string.Format(CultureInfo.InvariantCulture, "{0} <{0}@{1}>", name, domain);
    return new MailAddress(email);
	
````

Basically, the `MailAddressGenerator` would create by hand two strings, one to represent the part before the '@' and one for the domain.  In order to ensure created email addresses couldn't slip out into the wild, the `ficitiousDomains` is simply an instance collection of safe domains (e.g. `example.com`, `example.org`, `example.net`).  However, in AutoFixture it is typically not good form to produce additional specimens when generating a certain type.  Instead, that should be deferred to the `ISpecimenContext` parameter.  In other words, something like the below would be preferable:

````c#

	var name = context.Create<SomeTypeRepresentingEmailName>();
	var domain = context.Create<SomeTypeRepresentingEmailDomain>();

````

The upsides to this approach:

- Deferring creation of those specimens back to the builder chain allows you to better customize those individual pieces
- It is easier to reason about `MailAddressGenerator` and its logic

Here's a key question:  since ultimately we need the string representation of our specimens, why don't we just ask the `ISpecimenContext` to resolve them as strings within `MailAddressGenerator`?  That would definitely work, but the risk is: suppose someone has customized string generation in AutoFixture to produce strings that result in invalid `MailAddress`es?  For that reason, we opted to use a signal type to represent the specimen we needed.  

First, in terms of names, an email address is made up of two parts: the local part and the domain, separated by the '@'.  Any cursory web search will reveal that a number of RFCs touch on the format of email addresses, but they roughly be described as:  between 1 and 64 characters in length; specific list of allowable special characters; only US ASCII characters.  That last one is fairly interesting.  Here are some local parts that Mark offered that presumably he's seen in the wild, but are prohibited per the RFC:

````c#

	[Theory]
    [InlineData("ndøh")]
    [InlineData("ndöh")]
    [InlineData("åhnej")]
    [InlineData("ñoñó1234")]
    public void LocalParts_AreInvalid(string localPart)
    {
        var pattern = @"^(?!\.)(""([^""\r\\]|\\[""\r\\])*""|"
                                        + @"([-A-Za-z0-9!#$%&'*+/=?^_`{|}~]|(?<!\.)\.)*)(?<!\.)$";
            
        var invalid = System.Text.RegularExpressions.Regex.IsMatch(localPart, pattern);
        Assert.False(invalid);    
	}
	
````

How could that be?  How could email addresses be in use that are invalid per the RFC?  Well therein lies the rub.  The RFC says one thing, but email vendors can (and do) allow addresses to be used that violate these rules and that is their prerogative.  And in some ways that's good because the RFC is pretty restrictive as-written with basically no support for non-US characters.  That is why the typical guidance you here regarding validating email addresses is:  don't.  Don't bother.  Perhaps you do some basic syntactic validation or maybe run some regular expressions to ensure there's nothing suspicious, but otherwise the only true way to know if an email address is valid or not is to send an email message to it.  

That pretty quickly put me into an interesting spot.  I pitched softening the rules a bit (such as to support non-US ASCII characters, but otherwise keep the rules as-is), at which point Mark guided me to a different conclusion.  How was AutoFixture planning to use the `EmailAddressLocalPart` specimen?  As shown, we plan to request one from the `ISpecimenContext`, combine it with a domain, and then construct a `MailAddress` from it.  So for our purposes, the thing that makes an `EmailAddressLocalPart` valid is simply that `MailAddress` is able to accept it.  `MailAddress` already has a lot of validation rules that it applies in the constructor, unfortunately these are done by a `MailParser` class, which is internal within the `System` assembly so we cannot directly access it (or at least not easily).  And while we could reverse its rules, we'd be forever subject to the whims of bug fixes and modifications to that logic where `MailAddress` is concerned.  For that reason, `EmailAddressLocalPart` has almost no validation in it (other than ensuring the local part constructor parameter not null and not empty) and `MailAddressGenerator` becomes:

````c#

	try 
	{

		var localPart = context.Resolve(typeof(EmailAddressLocalPart)) as EmailAddressLocalPart;
		
		if (localPart == null)
		{
			return new NoSpecimen(request);
		}
		
		var domain = this.fictitiousDomains[(uint)name.GetHashCode() %3];
		
		var email = string.Format(CultureInfo.InvariantCulture, "{0} <{0}@{1}>", localPart, domain);	
		
		return new MailAddress(email);
	}
	catch (ArgumentException)
	{
		return new NoSpecimen(request);
	}
	catch (FormatException)
	{
		return new NoSpecimen(request);
	}
	
````

Since we're asking the `context` for an `EmailAddressLocalPart`, it needs to able to resolve it so we also have an `EmailAddressLocalPartGenerator`, which resolves a string from the `context` and produces `EmailAddressLocalPart` from it.  

<h2>What If I've Modified AutoFixture's String Generation?<h2>

Let's say you're the person that has modified AutoFixture's behavior where string generation is concerned and you're also trying to use its behavior for `MailAddress`es and it's breaking - what should you do?  In case you're not familiar with AutoFixture's design, one can separate the builders it applies into two processes.  First, it has a collection of builders that it attempts to use to satisfy type requests.  These builders are in the `Customizations` property on the `Fixture` instance itself.  When AutoFixture gets a request to create a type, it first checks to see if any of these builders can satisfy it and if they do, it uses those values.  If none of the custom builders can handle the request, AutoFixture next uses its `Engine` to resolve the types.  The `Engine` is a separate collection of builders and these are what you could consider the core AutoFixture builders that most everyone relies on without thinking about it.  The generators I discussed above, `MailAddressGenerator` and `EmailAddressLocalPartGenerator`, are both hooked into the `Engine` automatically so are always available.  But you as the user are free to define your own builder that will resolve `MailAddress` or `EmailAddressLocalPart` however you see fit to work around a case when string generation causes mail address validation failures.  This seems fairly unlikely, but given the goal of AutoFixture being widely applicable to many situations, it is set up to support these needs, albeit you take on a bit more of the burden yourself.  


