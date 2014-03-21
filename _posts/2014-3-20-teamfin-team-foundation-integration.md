---
layout: post
title: TeamFin - Team Foundation Integration
---
##{{ page.title }}
_{{ page.date | date: "%-d %B %Y" }}_

Continuing my week of first (after my first blog post), I've uploaded my first repository into GitHub.  I've contributed some minimal items to some projects, but this is the first one under my own name, which is pretty exciting.  

I've named it TeamFin, short for Team Foundation Integration.  There are a few similar packages I've seen out there, but none that seemed quite what I was after.  At work, my boss mentioned to me "It would be great if we could have something we could run after someone creates a new work item in TFS that automatically creates some other work items."  So the most common example is if we create an Epic, we will generally also create an Architectural Review item to it.  There are very few Epics that don't undergo such a review so the point was to save some time and ensure all items receive their child tasks consistently so I volunteered to take it on.

At the time, I'd already been looking into some TFS integration because I was attempting to use the API as an easier way to modify our automated build definitions.  I'll save that for another time, but every 6-8 weeks (i.e. our release schedule), we flip our development environment so we typically update our service builds to the new environment.  I'd grown tired of doing it by hand so I was working on a small utility to do it for me.  

I was somewhat impressed with how relatively painless the TFS API was around the builds, but I spoke too soon because after I volunteered to take on this TFS task, I immediately realize that querying work items from TFS was calling a method called Query that took a single string argument, this very specific TFS schema query (a work item query).  

Since LINQ to SQL exists solely to handle this sort of case, it took no real thought to decide to implement basically LINQ to TFS.  I'd been curious about doing this anyway, but now was about as good a time as I'd ever have.  

Matt Warren has a 7-year old tutorial on writing a [custom Query Provider](http://blogs.msdn.com/b/mattwar/archive/2008/11/18/linq-links.aspx) which provided a good start, if far beyond what I needed to do.  

Despite my boss' instructions on what needed to be done (work item creation), I felt that a key feature was the ability to identify items that were missing specific child items (such as Epics that didn't have Architectural Review items attached) so I worked first on getting the querying features functional.  The most tedious part was figuring out the TFS schema as there isn't much documentation. Ultimately the easiest route was to just dump every field in the schema into text files (programmatically), then look through those to determine the underlying field names and allowed values.  Once that was done, it was simply a matter of crafting a Visitor to walk an expression against my custom TfsWorkItem (a wrapper around the native WorkItem) and generate the SQL.  

There's plenty of features left to implement, but I look forward to adding to it over the next few months if only as an exercise for myself.  If others stumble across it and find any value in it, even as a reference, that would be great too.
