---
layout: post
title: Getting Started with Deedle for Machine Learning
tags: machine learning
description: Deedle is an awesome data frame library for .NET that I don't think has the exposure it deserves. This post explores some of the basics of Deedle to smooth out the initial learning curve applied to a basic machine learning classification problem.
---
Deedle is a fantastic data frame library available in both C# and F#, but I don't know if it's earned the exposure it deserves. One explanation for that is machine learning in .NET has never been terribly well supported, but as I've investigated that further, there are some key libraries that make it not just possible, but a great experience.  

So what is a data frame? Picture this situation.  You have a CSV file where each row is a different observation.  Each attribute within the row corresponds to an attribute.  For explanation purposes, we'll use the Wine classification dataset available from the [UCI machine learning repository](http://archive.ics.uci.edu/ml/datasets/Wine).

In typical .NET usage, you might simply make a custom type (`WineCultivar` or some such) and then use a StreamReader to read lines from the file, spilt it, and then generate instances of your class.  Net result: IEnumerable<WineCultivar> ready for your use.  However, machine learning tasks require a fair amount of data manipulation and pre-processing to prep the data for learning.  For example: you may need to centralize or standardize continuous data; you may need to explode categorical attributes into binary ones; you may need to impute missing values.  That becomes a rather difficult process now as WineCultivar doesn't support that.  We likely end up using a lot of anonymous types or dictionaries to handle the various transforms and while that can work, it's ugly.  Too, suppose we want to operate on data at the column level (such as center all values in the row by subtracting the mean and dividing by the standard deviation)?  There's no particularly clean way to do that.  Enter data frames!

<h2>Data Frames</h2>

Data frames exist in a lot of different languages.  Python has the awesome Pandas library and R gets maximum leverage out of them.  Imagine we have a way to view our data in a tabular fashion (rows and columns), but that allows each column to be heterogeneous.  In other words, we may want some columns to be text and some to be integers and some to be floating-point.  In a multi-dimensional array, we could only achieve that by declaring it of type object[,] and that's so general as to be useless.  Deedle, at a high level, gives us that power and flexibility to slice, index, group, and select data in a row-wise or column-wise fashion and it keeps track of type conversions in a much cleaner way.  

<h2>Loading a Data Frame</h2>

Terminology is important in Deedle.  The documents are excellent, but my goal is to make them even easier.  First, both columns and rows have keys.  In the default usage, the row keys will be integers (auto-incrementing ones technically) and columns are strings.  Here's how we can construct a Frame<int, string> by reading in the wine.data CSV file. 
<br/>
````c#
var frame = Deedle.Frame.ReadCsv(filename, hasHeaders: false);
````
<br/>
In this case, Deedle defaulted that we wanted an int row key and string column key.  Since our input file does not contain headers, we tell Deedle that as well through the method argument.  Deedle thus uses Column_0, Column_1, etc as the column names.  However, we actually know the column names, they're just in a separate file.  So we can easily set the column names after the fact by doing:
<br/>
````c#
frame.RenameColumns(new string[]
{
    "Label", "Alcohol", "Malic Acid", "Ash", "Alcalinity of Ash", "Magnesium", "Phenols", "Flavanoids",
    "Nonflavanoid Phenols", "Proanthocyanins", "Color Intensity", "Hue", "OD280/OD315", "Proline"
});
````
<br/>
<h2>Changing Values Within A Column</h2>

For machine learning purposes, `Label` is the class label for the observation and the other columns are the features.  Under the hood, Deedle is storing the data in a column-centric fashion.  What I mean by that is, you can picture the Frame as being a collection of collections.  Each sub-collection corresponds to one of the features (including the `Label` one).  While row access is supported in Deedle, the preferred way to slice and access data is through the columns as typically that is a more natural use-case for a data frame.  If you find yourself accessing through rows a lot, you can interchange the structure by using `Transpose`.  

In this data set, there are 3 possible class labels (1, 2, or 3).  This is a good fit for multi-class classification and we're going to use the Accord.NET library for this.  I will not go into much detail on that library and will save that for a follow-up post.  One route to go for this problem, particularly knowing that 2 of the classes are not linearly separable, is to try to use a support vector machine with a nonlinear kernel.  Accord provides such an implementation of those as well as various learning algorithms to train the SVMs, but here's the challenge: Accord requires the labels in a multi-class scenario to start at 0 and go up: our labels as given in the data set don't conform to that.  Deedle can handle that with minimal effort:
<br/>

````c#
var relabeled = frame.Columns["Label"].Select(kvp => (int)kvp.Value - 1);
frame.ReplaceColumn("Label", relabeled);

````
<br/>
The key thing to know is that Series (the columns) are immutable.  All we can do is generate new series from the old series and then replace them.  However, Frames are to a limited extent mutable.  In this case, we want to subtract 1 from every label to move them from the range [1,3] to the range [0,2]. 

To do so, we simply choose the `Label` column using the Columns property, then use a normal LINQ projection.  The trick in this particular usage is that we are going to enumerate through KeyValuePairs where the key is the row key and the value is the value in the Series.  In this case, we don't need the key at all; all we need to do is subtract 1 from the value.  We have to cast it to an int because under the hood, Deedle is storing the data as objects so we have to keep track of this fact (though there are some ways to be safe about it we'll cover another time).  Now `relabeled` is of type `Series<int, int>` where the first type parameter is the type of the row key (unchanged) and the second parameter is the type of data stored within the series (changed to an int by us).  Now what we need to do is replace the previous `Label` column in the frame with our new version, which can be done easily enough using ReplaceColumn.  This mutates the data frame accordingly.  The one caveat here is that Deedle will now have the `Label` column at the "end" of the frame instead of the beginning so if you inspect or print it, you'll see it shows up in a different place. 


