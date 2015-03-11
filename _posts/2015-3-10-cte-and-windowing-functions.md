---
layout: post
title: Basics of Common Table Expressions and Windowing Functions
tags: SQL
description: A not as well known set of features in SQL Server, Common Table Expressions (CTE) and windowing functions can help solve strange SQL problems.
---
Most developers are aware of how to use basic SQL constructs (Select, Update, Insert) even if their level of database programming is very minimal.  I was fortunate that before I was a fulltime developer, I worked in a role that required me to read, write, and troubleshoot an intense amount of SQL (and actually wasn't a DBA job).  Over the course of time, I became pretty fair at SQL and others used to bring me random SQL problems for help.  Most of these were straightforward enough problems, but occasionally I would get one that was really difficult, and I really enjoyed those.  Last night a current colleague of mine emailed me about a problem he was having doing some integration work and the solution was a set of SQL Server features that I think deserve to be talked about more:  common table expressions (CTE) and windowing functions.  There's a lot of power there and it can be overwhelming so the point of this post is to simply describe the problem and then illustrate how CTE and windowing functions made the solution pretty trivial.  

Here's the problem:  we have a table of data about employment applicants (i.e. people who have applied for jobs) in a legacy system.  We're moving to a new system so we need to take the data from the first system and transform it into a format required by the second system so it can be imported.  

First, here's the data in the legacy system in a single table (greatly stripped down for illustrative purposes, the real one has about 250+ columns):

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

We'll call Name, Phone, and EyeColor the "key" fields.  That's a bit of a misnomer since these keys don't uniquely identify a single record, but I'm using the term simply to mean that those values are the same for a single person.  We need to sequentially number each record with the same key (and we'll call it PositionCount).  Franklin has two records so the first is 1 and the second is 2, but Hoolihan has only a single record so its sole record is 1.  We only want to print the key fields for the first position record; otherwise we want to print blanks for the key fields, but still print the position data.  Here's how we want the data to look for loading into the target system.  

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

The first part to tackle:  how can we determine the position counts?  Most people can instantly think of an imperative solution (just loop through the records, adding an incrementing value to that column, resetting for each person), but in SQL we want to think relationally.  A `Group By` would let us group the records by our keys, but at best we could obtain a count of the records in the group, not quite what we want.  It's here that a windowing function really shines.  

At the most basic level, a windowing function lets us define a partition (which is similar to a group) against which we can execute some statement.  Below is the SQL that will do the job for me:

````sql

select *, row_number() over (partition by [Name] order by [Name] asc) as [PositionCount] from Persons)

````

Probably not terribly hard to understand, but the `OVER` function is the windowing function.  We instruct it to partition our set based on `Name`, then order it by `Name` (which is required), then apply SQL Server's row_number() function within each partition.  This will have the effect we want: the first record in each partition will get the value 1, the next 2, and so on, but the counts reset for each partition.  By modifying the partition columns, you can control to what level the records are numbered.  See the documentation on the  <a href="https://msdn.microsoft.com/en-us/library/ms189461.aspx">OVER function on MSDN</a> to learn more.   The shown SQL is now capable of providing this result set:

<table class="sql">
<tr><th>Name</th><th>Phone</th><th>EyeColor</th><th>PositionID</th><th>Title</th><th>PositionCount</th></tr>
<tr><td>Franklin</td><td>1212</td><td>Blue</td><td>123</td><td>General Manager</td><td>1</td></tr>
<tr><td>Franklin</td><td>1212</td><td>Blue</td><td>67</td><td>Salesman</td><td>2</td></tr>
<tr><td>Pierce</td><td>1313</td><td>Blue</td><td>234</td><td>Digger</td><td>1</td></tr>
<tr><td>Pierce</td><td>1313</td><td>Blue</td><td>456</td><td>Porter</td><td>2</td></tr>
<tr><td>Hoolihan</td><td>1414</td><td>Green</td><td>123</td><td>General Manager</td><td>1</td></tr>
<tr><td>Bob</td><td>1515</td><td>Blue</td><td>86</td><td>Maid</td><td>1</td></tr>
<tr><td>Bob</td><td>1515</td><td>Blue</td><td>90</td><td>Electrician</td><td>2</td></tr>
</table>

From here, what we want to achieve is conceptually pretty easy:  we only want to print the value of the key fields when `PositionCount` is 1, otherwise we want to print a blank (or null).  We could of course make a new table (either temporary or real) and put our newly formed result set in that, then operate against it, but a common table expression (a CTE) is for me faster to develop (albeit with some caveats I'll cover at the end). 

What is a common table expression (CTE)?  Without cribbing directly from Microsoft on the subject, it's in many ways an in-memory view.  You provide it with a name and you can thereafter use that name for the result set, same as you would a query.  The CTE can be used recursively in other queries and combined in some very powerful ways.  They are incredibly useful if you'll be referencing the same table or result set multiple times within a query as you can apply a name to that set and just use the name.  And due to how they're defined, they make queries a lot easier to read as they basically build from the top down in an organized fashion, which is genearlly far easier to decipher than a query with a lot of nested queries. 

A CTE is defined in either of the forms:

````sql

WITH (CTE Name> AS (<query definition>) 

WITH <CTE Name> (<column names>) AS (<query definition>)

````

The first form defines a CTE with the given name and its columns will be all the columns from the query definition.  The second allows you to specify the column names you want available in the CTE; these are drawn from the query definition.  Only those columns defined in the CTE are then avaiable.  So what's the point of the CTE?  You can then use it like a table: 

````sql

with PositionsCounted as (select *, row_number() over (partition by [Name] order by [Name] asc) as [PositionCount] from Persons)
select * from PositionsCounted

````

The above SQL is equivalent to just executing the query definition directly, all we've really done at this point is given the result set a name we can operate against.  But for the final piece, we can do this to produce our final result set:

````sql 

with PositionsCounted as (select *, row_number() over (partition by [Name] order by [Name] asc) as [PositionCount] from Persons)
select case when [PositionCount]>1 then '' else [Name] end,
	   case when [PositionCount]>1 then '' else [Phone] end,
	   case when [PositionCount]>1 then '' else [EyeColor] end,
	   [PositionID], [Title], [PositionCount] 
	   from PositionsCounted

````

The trick to this SQL is that for each of our key fields, we inspect the value of `PositionCount` and that determines what we should output (either the value of the column or a blank).  Otherwise, we simply want the non-key fields returned as-is.  Obviously this approach is difficult to scale if there are hundreds of key fields (as each would have to be added manually as a case expression), but in this case when there's only a minimal number of key fields, it's trivial. 

CTEs can be combined as well, such as the below, which simply puts the output of the first CTE into the second, a trivial but hopefully illustrative example:

````sql

with PositionsCounted as (select *, row_number() over (partition by [Name] order by [Name] asc) as [PositionCount] from Persons),
OutputReady as 
(select case when [PositionCount]>1 then '' else [Name] end [Name],
	   case when [PositionCount]>1 then '' else [Phone] end [Phone],
	   case when [PositionCount]>1 then '' else [EyeColor] end [EyeColor],
	   [PositionID], [Title], [PositionCount] 
	   from PositionsCounted)
select * from OutputReady

````


So we've seen some benefits of CTE, what are the downsides?  First, they are only in scope for a single query.  If you want to reuse them, they have to be copied and pasted into all places that need to reference them.  If you have such a need, temp tables or something more permanent (or semi-permanent) is a better bet.  Second, because they do not exist in tempdb, you cannot add indexes or anything that would improve the performance (and constraints and stats are likewise not available).  Again, if that's your need, a temp table would be more in order.  

I hope you've enjoyed this basic look at common table expressions and windowing functions in SQL Server.  This has only scratched the surface of these features so I definitely suggest working through some examples to get a feel for how and when each can be applied in real life situations.  




