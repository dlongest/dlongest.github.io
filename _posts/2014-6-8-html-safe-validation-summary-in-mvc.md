---
layout: post
title: HTML-Safe Validation Summary in MVC
---
##{{ page.title }}
_{{ page.date | date: "%-d %B %Y" }}_

My team at work is writing an MVC application for customer login and various related self-service tasks.  This application replaces a legacy ASP.NET application that does a similar job, but we're switching vendors (so moving from an identity platform hosted on-premises to a different company's platform in the cloud).  I last used MVC about 2 years back and I've been impressed with the improvements in MVC 5 over MVC 3 from back then, so much of it seems to just work in a way that I wrestled with a lot back then.  Although it's certainly possible I've just come a long way in my understanding of the technology and it was never MVC's fault I used it wrong. 

One thing that still seems a lot harder than it should be is validation and error messaging.  MVC has all the built-in jQuery validate and unobtrusive tools driven from the various .NET annotations and that generally works great.  But at least for us, the default annotation error messages are not sufficient.  Also, it would be better to put them all in one place for maintenance purposes (and ideally, have them externalized).  Also, we want the front-end and back-end to cause them to look the same in case users have Javascript disabled.  Now I should say that our enterprise site doesn't work for you if Javascript is off so I'm not exactly sure why we're pushing this part so hard as I'm also not aware of plans to make our site work without it.  But nonetheless, we march forward. 

Usually the first realization around this type of messaging is to just put them into a Resources file.  This achieves our goal of putting them in one place, but they're not externalized as resources are compiled into the assembly.  Nevertheless, a good first step.  So here's an example of how resources look on the annotation if you haven't seen it before:

```csharp
		[DataType(DataType.Password)]
        [Required(ErrorMessageResourceType = typeof(ControllerResources), 
				  ErrorMessageResourceName = "Login_PasswordRequired")] 
        public string Password { get; set; }
```

Simple enough, both the front-end and the back-end can use the message out of the resources.  However, let's go one step forward.  As part of our UX guidelines, we want to show the validation messages in a single box at the top of the page (so basically Html.ValidationSummary).  But one more wrinkle, we want the property names to be bolded within the summary.  So in our example above we would want to see ""**Password** is required" or whatever our specified text is.  So therein lies a separate new difficulty.  

The front-end code could handle it as long as we minimally follow some convention.  Another way is: why can't we just specify the HTML for the resources themselves and have that passed forward into the markup?  Yes, plenty of reasons why that might not be great, but since these are maintained by developers anyway, not a bad preliminary step for us.  But Html.ValidationSummary will URL-encode our resources and they will not have our intended effect.  To that end, I authored a `HtmlSafeValidationSummary` that does the job for us.  It's an HTML helper that walks through the model state errors and uses the `TagBuilder` class to concatenate all of them into an MvcHtmlString, thus avoiding the encoding.  See the [HtmlSafeValidationSummary gist](https://gist.github.com/dlongest/eef964791dacf8c2c4ed).  

This relies on the fact that are HTML-ified messages have already been loaded into the model state via the annotations.  A more dynamic approach would be to not have to put the HTML markup into the resources at all, but have our custom helper match names within the error message with the property names and apply the bolding as necessary.  Long term we may end up there, but lot of features to go and the above came together in less than an hour.  Credit to Phil Haack who covered [an approach like this](http://haacked.com/archive/2011/07/14/model-metadata-and-validation-localization-using-conventions.aspx/) on his blog years back.  




