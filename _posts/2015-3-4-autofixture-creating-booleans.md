---
layout: post
title: Creating Booleans in AutoFixture
tags: autofixture
description: Since I've seen this bite some developers recently, a quick discussion on AutoFixture's boolean creation logic.
---
I've gotten some developers at work into unit testing now, which is great, but they're going through some of the inevitable growing pangs.  One thing I saw a developer experience is related to how AutoFixture creates booleans.  It's not overly complicated logic (I mean there's only two values), but since I've seen it cause some grief, thought it was worth describing. 

AutoFixture's goal is to produce anonymous values for unit testing.  For more on why this is a good goal, check out the <a href="https://github.com/AutoFixture/AutoFixture">AutoFixture docs</a>.  For our purposes though, an anonymous value is one that we need to provide as part of our test, but we don't actually care about the value.  Let's take as an example an entity we'll call a Grader (a person that grades exams).  Here's an implementation we'll use:

````csharp

	public class Grader
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public bool IsActive { get; set; }
    }
	
````

Now suppose we're testing a component whose logic is based on IsActive being true or false.  We don't otherwise care about the value of FirstName or LastName, but let's further suppose they do need to be supplied.  Well of course we could do something like:

````csharp

	[Fact]
	public void WorksCorrectly_WhenGraderIsActive()
	{
		var grader = new Grader 
		{
		    FirstName = "Don't need", "LastName = "Don't need", IsActive = true 
		};
		
		/// Call SUT and verify
	}

````


But having to generate values like this is a waste of time, hence AutoFixture:

````csharp

	[Fact]
	public void WorksCorrectly_WhenGraderIsActive()
	{
		var grader = new Fixture().Build<Grader>().With(a => a.IsActive = true).Create();
		grader.IsActive = true;
		
		/// Call SUT and verify
	}

````

I'll gloss over alternate ways to customize the Grader such that IsActive ends up being true, but the above leads us to an object where FirstName and LastName have values, but they're meaningless (for string properties by default the value is the name of the property and a GUID).  What if I didn't specify IsActive = true, what would be the value?  Interestingly here, AutoFixture will always use `true` for the first boolean that gets created.  However, it will alternate between true and false for each boolean it creates as shown in the below test.  

````csharp

	[Fact]
     public void Fixture_CreatesDifferentBooleans_OnConsecutiveCreations()
    {
		var fixture = new Fixture();

        var first = fixture.Create<bool>();
        var second = fixture.Create<bool>();

        Assert.NotEqual(first, second);
        Assert.True(first);
        Assert.False(second);
    }
	
````

This governs any booleans that AutoFixture creates both directly and as part of an object.  In the below test, we create an intervening Grader and its `IsActive` property gets the value false, ensuring that `first` and `second` are true. 

````csharp

    [Fact]
    public void Fixture_CreatesSameBoolean_AfterInterveningBooleanCreation()
    {
         var fixture = new Fixture();

         var first = fixture.Create<bool>();
         var grader = fixture.Create<Grader>();
         var second = fixture.Create<bool>();

         Assert.Equal(first, second);
         Assert.True(first);
         Assert.False(grader.IsActive);
    }
	
````

The final test below won't be a surprise, but boolean properties on the same object will also receive alternating values:

````csharp

    [Fact]
    public void Fixture_AlternatesBetweenTrueAndFalse_ForBooleanProperties_OnSameInstance()
    {
        var fixture = new Fixture();
            
        var booleans = fixture.Create<Booleans>();

        Assert.NotEqual(booleans.First, booleans.Second);
        Assert.True(booleans.First);
    }
		
````

Again, this is far from rocket science but since it came up recently I thought I would just briefly document it so at least in the future I could point someone here for an explanation.


