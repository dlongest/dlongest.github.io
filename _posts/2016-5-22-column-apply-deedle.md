---
layout: post
title: Column Apply in Deedle
tags: machine learning
description: This post demonstrates how to use ColumnApply in Deedle particularly as it pertains to what columns are selected for application.
---
The concept of `Apply` is pretty universal in data frame libraries.  At a glance, it allows an operation to be performed on every element within a given subset (typicaly either all elements in a column or all elements in a row).  Conceptually this is just a loop, but depending on the parameterization, the frame takes care of some other messiness as well.  In Deedle, `ColumnApply` allows you to invoke some logic on every column of a certain type.  The challenge, though, is how to use it effectively when some columns are convertible to other types (i.e. most of your columns are double, but some are int and that is a trivial conversion in .NET).  Deedle's `ColumnApply` supports some parameters to address that point and that will be demonstrated in this post.

All examples will use the below data frame that emulates a dataset with data points in a variety of data types. 

````c#
 var frame = Frame.FromRecords(new[] {  new { Label = 1, RealAttr = 2.0, IntAttr = 1, BoolAttr = true, StringAttr = "a" },
                                                new { Label = 1, RealAttr = 3.1, IntAttr = 2, BoolAttr = true, StringAttr = "bb" },
                                                new { Label = 2, RealAttr = 0.9, IntAttr = 1, BoolAttr = false, StringAttr = "ccc" },
                                                new { Label = 2, RealAttr = 0.4, IntAttr = 2, BoolAttr = false, StringAttr = "dddd" },
                                        });
````

In all cases, the function we're going to apply simply doubles each value in the column. 

<h2>Exact</h2>

Exact is precisely what it sounds: unless the type of the column is an exact match for the type parameter to `ColumnApply`', the column will be skipped.  Note that `ColumnApply` mutates the frame so we have to either overwrite our `frame` reference or assign it to a new value (here I just call Print() on the new reference and then it's lost):

````c#
frame.ColumnApply<double>(ConversionKind.Exact, series => series * 2).Print();
````

<table><tr><td></td><td>Label</td><td>RealAttr</td><td>IntAttr</td><td>BoolAttr</td><td>StringAttr</td></tr>
<tr><td>0 -></td><td>1</td><td>4.0</td><td>1</td><td>True</td><td>a</td></tr>
<tr><td>1 -></td><td>1</td><td>6.2</td><td>2</td><td>True</td><td>bb</td></tr>
<tr><td>2 -></td><td>2</td><td>1.8</td><td>1</td><td>False</td><td>ccc</td></tr>
<tr><td>3 -></td><td>2</td><td>0.8</td><td>2</td><td>False</td><td>dddd</td></tr>
</table>

As you can see, `RealAttr` has each of its values doubled, but all other columns are unaffected.

<h2>Flexible</h2>

Flexible conversion is intended to make maximum use of .NET type conversions through the static `Convert` class and similar methods. Its use in `ColumnApply` has an interesting effect:

````c#
frame.ColumnApply<double>(ConversionKind.Flexible, series => series * 2).Print();
````

<table><tr><td></td><td>Label</td><td>RealAttr</td><td>IntAttr</td><td>BoolAttr</td><td>StringAttr</td></tr>
<tr><td>0 -></td><td>2</td><td>4.0</td><td>2</td><td>2</td><td>a</td></tr>
<tr><td>1 -></td><td>2</td><td>6.2</td><td>4</td><td>2</td><td>bb</td></tr>
<tr><td>2 -></td><td>4</td><td>1.8</td><td>2</td><td>0</td><td>ccc</td></tr>
<tr><td>3 -></td><td>4</td><td>0.8</td><td>4</td><td>0</td><td>dddd</td></tr>
</table>

As you can see, a lot more columns are affected this time than when using `Exact`.  Both `Label` and `IntAttr` have had their values doubled in addition to `RealAttr`.  The really interesting one, though, is that `BoolAttr` has gone from boolean representation to integer. 

<h2>Safe</h2>

Safe is the intermediate level between `Exact` and `Flexible`: it will allow numeric widening conversions, but no others:

````c#
frame.ColumnApply<double>(ConversionKind.Safe, series => series * 2).Print();
````

<table><tr><td></td><td>Label</td><td>RealAttr</td><td>IntAttr</td><td>BoolAttr</td><td>StringAttr</td></tr>
<tr><td>0 -></td><td>2</td><td>4.0</td><td>2</td><td>True</td><td>a</td></tr>
<tr><td>1 -></td><td>2</td><td>6.2</td><td>4</td><td>True</td><td>bb</td></tr>
<tr><td>2 -></td><td>4</td><td>1.8</td><td>2</td><td>False</td><td>ccc</td></tr>
<tr><td>3 -></td><td>4</td><td>0.8</td><td>4</td><td>False</td><td>dddd</td></tr>
</table>

You'll see that again `Label` and `IntAttr` are affected, which makes sense as those are basic widening conversions.  However, `BoolAttr` is left alone and retains its boolean representation. 

<h2>Wrap Up</h2>

I hope that better illustrates the uses of `ColumnApply`.  Unfortunately within the lambda you supply, there is no way to determine exactly what column is being applied against so you cannot do any kind of conditional apply (i.e. apply to all floating point and integer columns except one named `Label` or something like that). 



