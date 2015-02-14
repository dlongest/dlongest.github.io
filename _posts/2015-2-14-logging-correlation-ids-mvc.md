---
layout: post
title: Logging Correlation Identifiers in ASP.NET MVC
tags: MVC, design
description:  An approach for managing and logging correlation identifiers for diagnostics and triage.
---
Over the months we've been writing a new login application against a cloud SaaS platform, we have slowly evolved our logging strategy for the application.  The evolution occurred because we needed more and more information for diagnostics and triage related to bugs (sometimes in our custom application, often in the vendor product too) and performance issues.  Some of the logging we employed that wrote into the event log on the login app server (of which there are 2) included:

<ul>
<li>Call times for the vendor services (SOAP and REST) as Windsor interceptors</li>
<li>Communication exceptions trying to reach the vendor services as Windsor interceptors</li> 
<li>Raw SOAP request and response messages as WCF behavior on the client proxy</li>
<li>Primary error page with reference ID shown to user as controller-view</li>
</ul>

These error messages have been incredibly useful.  The call times have driven a lot of performance testing and improvements that have been put in place (coupled with a separate application I wrote to crunch the call times, which I might present separately).  The raw SOAP request/responses have been useful because many bugs come down to the vendor asking what was returned by their system (presumption being we were handling it wrong), but the responses often showed a "smoking gun" that the response was missing key information.  

Recently as we've fought hard to move to deployment, I realized that there was a bit of an oversight in the logging:  while it wasn't too hard to trace backward through the log to identify a series of messages within a given use-case, it isn't the kind of thing anyone could do without a lot more knowledge of the platform calls (knowledge my team had gained over the last 9 months).  But there seemed conceptually a pretty simple solution to this problem:  all we needed was a way to correlate the log messages for a user within a single session.  I'm not even interested in tracking the user across multiple different sessions (although if that happens, fine).  I principally just want a way that if creating an account requires transitioning through 3 or 4 screens and that causes 8-10 log messages (with everything turned up), we can tie those log messages together as a matter of rote.  

Obviously lots of ways to do this but the below will present one way I've gone about it.  First, we have in place this logging interface:

````csharp

  public interface ILog
    {
        void Log(string message);
        void Log(string message, Exception ex);
        void Log(string message, object obj);
        void Log(string message, Exception ex, object obj);
    }
	
````

This interface is actually just a wrapper around an in-house logging library we have; this one closes over that type (which requires basically an event ID be provided) so that components in the application can be agnostic to the event ID they're using, which is configured by Windsor in the root.

That yields this basic implementation that we inject throughout the application to components that are logging-aware:

````chsarp

public class EventIdEnclosingLogger : ILog
    {
        private ILogger<object> logger;
        private EventId eventId;

        public EventIdEnclosingLogger(ILogger<object> logger, EventId eventId)
        {
            this.logger = logger;
            this.eventId = eventId;
        }

        public virtual void Log(string message)
        {
            this.logger.Log(message, this.eventId);
        }

        public virtual void Log(string message, object obj)
        {
            this.logger.Log(message, obj, this.eventId);
        }

        public virtual void Log(string message, Exception ex)
        {
            this.logger.Log(message, ex, this.eventId);
        }

        public virtual void Log(string message, Exception ex, object obj)
        {
            this.logger.Log(message, ex, obj, this.eventId);
        }       
    }
	
````

So given that I have a logging abstraction, what's an easy way I can introduce a a correlation ID into the mesages?  A decorator!  In case you're not aware, a decorator pattern allows the addition of functionality on top of an existing component.  So from the above types, I could define:

````csharp

	public CorrelatingLogger : ILog
	{
	
		private readonly ILog logger;
		
		//// elided
	}
	
````

So an ILog implementation is injected into the CorrelatingLogger, which can then enrich the parameters before calling into the inner logger implementation.  Mulitple layers of decorators can be applied, with the innermost implmentation finally doing the job.  But given our use of Windsor and interceptors, an interceptor was again the natural choice here.  That doesn't really alter the philosophy being employed here at all, just leverages the Windsor infrastructure.  

So this part probably makes sense, but I haven't answered the key question:  where am I getting the correlation IDs from and how will those be managed?  We could imbue the logging interceptor with the ability to manage the correlation IDs for us, but that feels like a single-responsibility violation: it should be responsible for computing appropriate log messages based on the correlation ID, not managing them.  Enter a new abstraction:  ICorrelationProvider.

````csharp

	public interface ICorrelationProvider
    {
        string GetCorrelationIdentifier();
    }
	
````

Simple enough, how will we power it?  Okay, now for the real debate.  First, the application is not backed by a database or store of its own (all of that being offloaded to the vendor stack) so no help there.  The most natural mechanism to pass state around during a single request is `HttpContext.Items`, which is a dictionary only available to the currently executing request (i.e. thread local, so there's no interference between other executing requests).  So that will work fine as storage during the request, but how will we ensure continuity of IDs across requests?  Simple enough we can use our good friend the cookie.  It's arguable we could/should have used Session, and that's a legitimate argument, but like I said, this is just one way to do it.  

We've now identified that we will use a cookie to manage the correlation IDs between requests and we will essentially load it into `HttpContext.Items` at the first opportunity during a request.  I considered overriding the `Initialize` method in the controller and having that manage the state for me, but that spread out knowledge of the correlation ID key to two places:  the controller and the correlation provider.  After initially implementing it that way, I realized that piece of knowledge had to be in one place so I switched to the below implementation.

A few notes for the code you'll see below:

<ul>
<li>We inject a lambda to provide us with the `HttpContextBase`, not the `HttpContext` itself since at the point when the provider is created, I have been burned by the context not being fully formed. Instead, we use the lambda as it's not until execution (when the context will be fully formed) that we actually need it.</li>
<li>We prioritize the value in the cookie over the one in Items so if there's a discrepancy for any reason, we allow the cookie to dictate whether we create a new ID (a GUID) and load it into Items or not.</li>
<li>If for some reason an exception is thrown while trying to create, obtain, or update the correlation ID, we will swallow it and return a hard-coded value.  This will let our system functionality continue unabated and make it obvious there's been an issue in the logging.  Hopefully though all such issues will be ironed out in development anyway, but we do not want the live functionality rendered unusuable simply because the log messages don't get this extra value.</li>
<li>GetCorrelationId violates CQS:  it both mutates system state (by adding/updating a cookie and items) and returns a value.  I felt the violation in this case was acceptable as it made the client code easier to follow.</li>
</ul>

````csharp

	public class HttpContextCorrelationProvider : ICorrelationProvider
    {
        private readonly Func<HttpContextBase> httpContext;
        private readonly string correlationKey;

        public HttpContextCorrelationProvider(Func<HttpContextBase> httpContext, string correlationKey)
        {
            if (httpContext == null)
                throw new ArgumentNullException("httpContext");

            this.httpContext = httpContext;          
            this.correlationKey = correlationKey;
        }

        public string GetCorrelationIdentifier()
        {
            try
            {
                var correlationId = GetCorrelationId();
                UpdateCookie(correlationId);
                UpdateContext(correlationId);
                return correlationId;
            }
            catch (Exception ex)
            {
                return "123456";
            }
        }
      
        private string GetCorrelationId()
        {
            var correlationCookie = this.Context.Request.Cookies[this.correlationKey];

            var existsInContextItems = this.Context.Items.Contains(this.correlationKey);

            if (correlationCookie == null && !existsInContextItems)
                 return Guid.NewGuid().ToString();

            if (correlationCookie != null)
                return correlationCookie.Value;
            else 
                return this.Context.Items[this.correlationKey] as string;
        }

        private void UpdateCookie(string correlationId)
        {
            if (CorrelationCookie == null)
            {
                this.Context.Response.Cookies.Add(new HttpCookie(this.correlationKey, correlationId));
            }
            else if (!CorrelationCookie.Value.Equals(correlationId, StringComparison.OrdinalIgnoreCase))
            {
                this.Context.Response.Cookies.Add(new HttpCookie(this.correlationKey, correlationId));
            }
        }

        private void UpdateContext(string correlationId)
        {
            var hasId = this.Context.Items.Contains(this.correlationKey);

            var currentValue = hasId ? this.Context.Items[this.correlationKey] as string : null;

            if (currentValue == null)
            {
                this.Context.Items.Add(this.correlationKey, correlationId);
            }
            else if (!currentValue.Equals(correlationId, StringComparison.OrdinalIgnoreCase))
            {
                this.Context.Items[this.correlationKey] = correlationId;
            }
        }     

        public HttpCookie CorrelationCookie
        {
            get
            {
                return this.Context.Request.Cookies[this.correlationKey];
            }
        }

        private HttpContextBase context;

        private HttpContextBase Context
        {
            get
            {
                if (context == null)
                    this.context = this.httpContext();

                return this.context;
            }
        }
    }
	
````

Finally we have our `CorrelatingLogInterceptor` that ties all these pieces together.  You'll see that the interceptor simply fetches the first argument (which is always the message for our `ILog` type), adds the correlation ID to it, updates the method argument, then calls the inner logger.  You'll notice that the interceptor does not currently guard that it has been applied to a correct invocation.  In other words, if you 
configured this interceptor against a type that isn't ILog, the implementation will not notice or care (until an exception gets thrown).  We rely on Windsor to properly use our `IModelInterceptorsSelector` that contains the logic to apply the interceptor (not shown). 

````csharp

 public class CorrelatingLogInterceptor : IInterceptor
    {       
        private readonly ICorrelationProvider provider;

        public CorrelatingLogInterceptor(ICorrelationProvider provider)
        {
            if (provider == null)
                throw new ArgumentNullException("provider");

            this.provider = provider;
        }

        public void Intercept(IInvocation invocation)
        {
            try
            {
                var message = GetMessageArgument(invocation);
                var correlationId = provider.GetCorrelationIdentifier();
                var newMessage = UpdateMessage(message, correlationId);
                UpdateArgument(invocation, newMessage);
            }
            catch (Exception ex)
            {
            }
            finally
            {
                invocation.Proceed();
            }
        }

        private string GetMessageArgument(IInvocation invocation)
        {
            return invocation.Arguments.First() as string;
        }

        private string UpdateMessage(string message, string correlationId)
        {
            return string.Format("{0}\n{1}: {2}", message, "Correlation ID: ", correlationId);
        }

        private void UpdateArgument(IInvocation invocation, string newMessage)
        {
            invocation.SetArgumentValue(0, newMessage);
        }
    }
	
````

This concludes the write-up on correlation event log entries across an MVC application using a Windsor interceptor and custom correlation provider on top of `HttpContext.Items`.  

