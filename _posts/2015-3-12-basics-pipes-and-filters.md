---
layout: post
title: Evolving to Pipes and Filters
tags: design
description: How to use a pipes and filters design to avoid massive OCP violations.
---
In a <a href="2015/03/10/cte-and-windowing-functions.html">prior post</a>, we looked at a pretty straightforward problem:  given data set representing job applicants, we wanted to massage it into a slightly different format for import into a new system and we examined how to use common table expressions and windowing functions to achieve it with pretty minimal effort.  This same problem lends itself quite nicely to a particular design pattern called pipes and filters so in this post we will explore that technique against this same problem.  

A quick refresher on the problem, we have a table that resembles the below:

<table class="sql">
<tr><th>Name</th><th>Phone</th><th>EyeColor</th><th>PositionID</th><th>Title</th></tr>
<tr><td>Franklin</td><td>1212</td><td>Blue</td><td>123</td><td>General Manager</td></tr>
<tr><td>Franklin</td><td>1212</td><td>Blue</td><td>67</td><td>Salesman</td></tr>
<tr><td>Pierce</td><td>1313</td><td>Blue</td><td>234</td><td>Digger</td></tr>
<tr><td>Pierce</td><td>1313</td><td>Blue</td><td>456</td><td>Porter</td></tr>
<tr><td>Hoolihan</td><td>1414</td><td>Green</td><td>123</td><td>General Manager</td></tr>
<tr><td>Bob</td><td>1515</td><td>Blue</td><td>86</td><td>Maid</td></tr>
<tr><td>Bob</td><td>1515</td><td>Blue</td><td>90</td><td>Electrician</td></tr>
</table>

We want to take that representation and from it, generate a file that looks like the below.  The key pieces:  the Name, Phone, and EyeColor columns should only print one time for a given person, otherwise should be blank; the positions each person has applied for should be sequentially numbered beginning at 1. 

<table class="sql">
<tr><th>Name</th><th>Phone</th><th>EyeColor</th><th>PositionID</th><th>Title</th><th>PositionCount</th></tr>
<tr><td>Franklin</td><td>1212</td><td>Blue</td><td>123</td><td>General Manager</td><td>1</td></tr>
<tr><td></td><td></td><td></td><td>67</td><td>Salesman</td><td>2</td></tr>
<tr><td>Pierce</td><td>1313</td><td>Blue</td><td>234</td><td>Digger</td><td>1</td></tr>
<tr><td></td><td></td><td></td><td>456</td><td>Porter</td><td>2</td></tr>
<tr><td>Hoolihan</td><td>1414</td><td>Green</td><td>123</td><td>General Manager</td><td>1</td></tr>
<tr><td>Bob</td><td>1515</td><td>Blue</td><td>86</td><td>Maid</td><td>1</td></tr>
<tr><td></td><td></td><td></td><td>90</td><td>Electrician</td><td>2</td></tr>
</table>

So how might we start this in C#? Well of course the most obvious way might be:

````c#

public class DataProcessor
{	
	public void Process(string connectionString, string outputFilename)
	{
		/// Read records from database into some collection, say IEnumerable<Applicant>
		/// For each Applicant, either project it into a new type that contains a "PositionCount" property or if it's already on Applicant, enrich it
		/// Write each Applicant record to a file	
	}
}

public class Applicant
{
	public string Name { get; set; }
	public string Phone { get; set; }
	public string EyeColor { get; set; }
	public int PositionID { get; set; }
	public string Title { get; set; }
	public int PositionCount { get; set; }
}

````

I've left out the code to do each piece, but it's probably exactly what you imagine.  One big thing stands out to me here:  calling this a DataProcessor is a bit of a code smell (much like calling something a `Manager`):  what are the steps involved in the processing?  What if they need to vary or are sometimes optional?  What if we want to reuse any single step in the process?  What if we want or need different input parameters to drive other processing steps?  We've guaranteed that we will perpetually be cracking open this class and tinkering with it to address most of these questions and that's a big open-closed violation.  

What's one easy heuristic to spot a potential OCP violation?  Give the method a hyper specific name.  So in this case instead of calling it `Process`', what if we named it `ReadFromDatabaseThenAddPositionCountThenWriteToFile`.  That would make it much clearer that this method is doing too much.  A-ha, you might say.  What if we just broke up the single Process method into separate methods within the class, problem solved!  Or is it?

````c#

public class DataProcessor
{	
	public IEnumerable<Applicant> LoadFromDatabase(string connectionString)
	{
		/// Read records from database into some collection, say IEnumerable<Applicant>	
	}
	
	public void AddPositionCount(IEnumerable<Applicant> applicants)
	{
		/// Compute the number of positions applied for by each individual person and sent Applicant.PositionCount appropriately. 
		/// Modifies the instances in the passed-in collection
	}
	
	public void WriteToFile(IEnumerable<Applicant> applicants, string absoluteFilename)
	{
		/// Write each Applicant record to a file	
	}
}

````

So now the use of our `DataProcessor` is fairly clear:

````c#
	
	var processor = new DataProcessor();
	var applicants = processor.LoadFromDatabase(/* string */);
	processor.AddPositionCount(applicants);
	processor.WriteToFile(applicants, "some_file.txt");

````

Yes, much clearer, but we still have some of the same issues we had before.  If we want to add more steps, we have to open up `DataProcessor` and add methods to it.  If we wanted to reuse any of the steps (were that possible), we can't do it cleanly.  And it's pretty evident here that single responsibility principle is being violated:  this class is doing at least 3 things and that list would only grow if more were added.  So what should we do?

If we agree that each of the methods in the current `DataProcessor` class is its own responsibility, what if we tried to find a design that put each of those in its own class?  Just shuffling the code around, we would have:

````c#

public class ApplicantLoader
{
	public IEnumerable<Applicant> LoadFromDatabase(string connectionString) { /* */ }
}

public class ApplicantEnricher
{
	public void AddPositionCount(IEnumerable<Applicant> applicants) { /* */ }
}

public class ApplicantWriter
{
	public void WriteToFile(IEnumerable<Applicant> applicants, string absoluteFilename) { /* */ }
}

````

Now we've cleaned up SRP and OCP violations quite nicely.  These classes decompose the problem into the distinct steps that previously were rather implicit (or at least coupled together) in the single `DataProcessor`.  But I think we can go one step further.  What's common about these three classes?  All of them work against a collection of Applicants.  And if we rewrite `ApplicantEnricher` as:

````c#

public class ApplicantEnricher
{
	public IEnumerable<Applicant> AddPositionCount(IEnumerable<Applicant> applicants) { /* */ }
}

````

Then we can quite clearly see there are 3 different types of activities going on here:  an initial source of Applicant records; a specific type of processing against the Applicant records that passes them forward; a sink that logically ends the processing chain in some way (such as writing to a file).  It is from this revelation that we can view this problem as befitting pipes and filters.  In this parlance a filter is a component that takes some input, processes it in some way, and sends forward either that same input or a different input.  A pipe refers to the connection between filters, but that doesn't always lend itself directly to a single component (so there may not be an `IPipe` abstraction necessarily). 

<img src="{{ site.url }}/images/pipes_and_filters.png"  />

One basic implementation of this pattern starts with a handful of interfaces:

````c#

public interface ISourceInput { }

public interface ISource<TIn, TOut> where TIn : ISourceInput
{
	TOut Load(TIn input);
}

public interface IFilter<TIn, TOut>
{
	TOut Filter(TIn input);
}

public interface ISink<TIn>
{
	void Sink(TIn input);
}

````

We define a marker interface `ISourceInput` that is used to tag any custom class we make that contains configuration or input parameters.  This isn't strictly required, but I do it to be slightly more explicit.  I find it generally true marker interfaces are a bit of a code smell.  Next the `ISource` interface is used to define our Source component.  Interestingly, since we've stipulated the source must take an input, `ISource` and `IFilter` differ only in that the former has a type constraint but again, I think there's value in being explicit here - nothing stops you from having the filter component also implement `ISource`.   A Sink can also be a Filter if one just ignores the output from the Sink.  We can now implement our previous components in light of these interfaces:

````c#

public class LoadApplicantSourceInput : ISourceInput
{
	public string ConnectionString { get; set; }
}

public class LoadApplicantsSource : ISource<LoadApplicantSourceInput, IEnumerable<Applicant>>
{
	public IEnumerable<Applicant> Load(LoadApplicantSourceInput input) { } 
}

public class ComputePositionCountForApplicantFilter : IFilter<IEnumerable<Applicant>, IEnumerable<Applicant>>
{
	public IEnumerable<Applicant> Filter(IEnumerable<Applicant> applicants) { }
}

public class WriteApplicantsToFileSink : ISink<IEnumerable<Applicant>>
{
	public WriteApplicantsToFileSink(string filename) { }

	public void Sink(IEnumerable<Applicant> applicants) { }
}

````

It's roughly the same code as we wrote initially, but we've simply split it up into components and raised the level of abstraction.  There's arguably a few strange things (why does the Source take an input for the connection string, but we provide the output file name to the Sink as a constructor argument) but any of those could be cleaned up or changed based on preference or the particular system they're being employed in.




