---
layout: post
title: Controlling collection name deserialization in .NET
---
##{{ page.title }}
_{{ page.date | date: "%-d %B %Y" }}_

It's not a thing I do often, but I found myself the other day needing to control XML serialization and deserialization.  Most of the time we just generate things like service proxies and don't alter the XML after that.  This case was a bit different as we have a vendor that is supplying us with an SAAS implementation for IAM and I'm managing the implementation on our side of an MVC application to handle login and some related identity services. 

After a few starts and stops on the vendor's part, we've finally figured out we're going to cause a "REST" service they'll expose to us.  I'm quoting "REST" because most of the time it seems like when people use that term, they mean transmitting data over HTTP, but are seldom (or never) even trying to address most parts of the REST pattern (such as hypermedia, resource design, etc).  Or basically, they're at Level 0 in the [Richardson Maturity Model]{http://martinfowler.com/articles/richardsonMaturityModel.html}. 

The service we're calling is for basic login:  send it a username and password, it sends back a success or failure and if success, a session.  The payload for the request and the response are XML.  We're using RestSharp for calling the service and that works fantastic, it reduces the number of lines of code by about 75% over using the HttpWebRequest class from the framework.  And the serialization of our request object into the XML payload worked fine out of the box.  But for some reason, I've been completely unable to get RestSharp to deserialize the response.  Or more specifically, I couldn't get it to deserialize a child collection.  So here are the response objects that exactly match the response schema:

```csharp
   public class loginResponse
    {       
        public string message { get; set; }       
        public string resultCode { get; set; }       
        public string sessionToken { get; set; }
        public List<response> authenticationResponses { get; set; }
    }

   
    public class response
    {
        public string Name { get; set; }       
        public string Value { get; set; }
    }   
```

This isn't rocket science and of course you could argue that there are much bigger fish to fry.  But it just bothers me to have classes with these names polluting the solution since a) the class names are incredibly generic; b) they don't conform to the .NET BCL guidelines (the vendor uses Java so these are more or less fine for Java conventions).  So what I'd really like to have is:

* loginResponse and response have names that actually convey a more specific usage 
* Each property should be cased appropriately
* The child collection property "authenticationResponses" should have a name that clearly associates it with the response class

I'll take a couple big steps right out of the gate.  First, the `XmlType` annotation allows us to specify the deserialization name for both classes and `XmlElement` does similar for the properties.  That lands us at:

```csharp
 [XmlType("loginResponse")]
    public class SaasIdentityLoginResponse
    {       
        [XmlElement("message")]
        public string Message { get; set; }

        [XmlElement("resultCode")]
        public string ResultCode { get; set; }  
     
        [XmlElement("sessionToken")]
        public string Session { get; set; }
        public List<SaasIdentityLoginResponsePair> authenticationResponses { get; set; }
    }

    [XmlType("response")]
    public class SaasIdentityLoginResponsePair
    {
        [XmlElement("name")]
        public string Name { get; set; } 
        [XmlElement("value")]
        public string Value { get; set; }
    }   
```

The final part is also the trickiest and that is ensuring our child collection is deserialized into the right property.  I think there's several ways to achieve this, but the one that ultimately worked for us was:

```csharp
		[XmlArray("authenticationResponses")]
        [XmlArrayItem("response", typeof(SaasIdentityLoginResponsePair))]
        public List<SaasIdentityLoginResponsePair> Responses { get; set; }
```

This lets us specify that the property is a collection named `authenticationResponses` and that it's a collection of XML schema elements called `response` that should be deserialized under our custom name. 

Simple enough, I'm sure most people know this, but I'm sure at some point I will need this magic voodoo again and I want to save myself the time it will take to figure it back out.  

