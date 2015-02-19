---
layout: post
title: Preventing Open Redirect Attacks in MVC - Part 1
tags: MVC, design
description: Part 1 of a design for preventing open redirect attacks in MVC will look at our custom login application's return URL and related rules for validating them. 
---
At work today some of us went through a "secure code training" as part of our PCI compliance efforts.  The training was led by a security consultant who previously did a secure code analysis on the login application my team has been building (the result of which:  2 minimal issues from static analysis that on investigation, were actually nothing).   His presentation was actually on the OWASP top 10 since that and PCI have some overlap.  I have some familiarity with that list since I've read some of <a href="http://www.troyhunt.com/">Troy Hunt's</a> writings on the subject plus watched his Pluralsight course <a href="http://www.pluralsight.com/courses/hack-yourself-first">Hack Yourself First</a> which demonstrates a lot of these attacks live via Fiddler.  So sidenote, if you're at all interested in the topic, definitely check out that course by Troy.  Anyway, one of the OWASP top 10 is called open redirect attacks and since I spent quite a lot of effort coding against that for the login application, thought the timing was good to explore that solution in more depth. 

<h2>What is an unvalidated forward or redirect attack?</h2>
Suppose you have a web page that performs some function and you take, as a querystring parameter, a URL that you will return the user to once the processing is completed.  Suppose that looks like something like:

````dosini

http://www.mysite.com/dosomething?Source=http://go.backhere.com

````

So in the above, after our page completes its work, it will send the user back to http://go.backhere.com.  Sound okay?  Well suppose the link I visit is:

````dosini

http://www.mysite.com/dosomething?Source=http://evil.site.com

````

Now after our work completes, we'll send the user to evil.site.com.  So the question now is:  well how on Earth would someone click such a link?  Well, like several other OWASP issues, it comes down to phishing.  Simply put, attackers will craft URLs in emails that look safe enough, but utilize this kind of open redirect to slip you over to a malicious site without you being aware of it.  And once that happens, all bets are off.  Read more about this issue <a href="https://www.owasp.org/index.php/Open_redirect">directly from OWASP.</a>

<h2>How do we prevent it?</h2>

You'll notice a theme in the OWASP list, but the solution is: input validation.  Do not blindly accept redirect URLs on your site, but compare them against a whitelist you maintain of safe redirects.  Any time the redirect would be to a URL not on the whitelist, simply don't perform the redirect.  That may mean substituting a safe redirect or taking the user to an error page but whatever it is, don't do it.  And that simple solution is the topic of this blog post as I spent quite a bit of time finetuning such a solution.  

The application I'm talking about provides enterprise login services against a cloud SAAS stack.  The application supports the basic identity functions you'd expect plus, of course, login.  But obviously a login application is not a terminal location:  users pass through it on the way to their resources (once properly authenticated and authorized of course).  So the first thing to establish is:  how do we know where to take users back to?  In our case, it comes from two places.  If the user attempts to access a resource that's protected by a login agent, the agent redirects the user to the application and provides a querystring parameter (TARGET) that specifies the absolute URI the user was attempting to access and where they should be returned after authentication.  Otherwise, the user can come directly to the login application to login.  Generally, that occurs when a user clicks the "Login" link on our enterprise site (powered by SharePoint) at which point SharePoint takes the user to the login application and provides the relative URI where they should be returned to in SharePoint.  In point of fact, the URI is of the form /_layouts/authenticate.aspx?Source=/some/location where /_layouts/authenticate.aspx is the SharePoint out of the box authorization page and Source actually provides it with instructions on where to take the user once it's done.  Very complicated, I know, but such is life.  

Anyway, so we have two different ways that we will be provided with the return URL for the user.  All we need to do is validate that it's safe, keep track of it, assist the user in logging in, and ultimately redirect the user to their return URL.  Let's take this in a few parts.  

<h2>Getting the Return URL</h2>

We previously acknowledged that the first request into the login application will typically provide the return URL for the user.  But at minimum, the user will be served a resource, input some values, and POST the form before the return URL will be needed.  And in our Create Account use-case, the return URL will come over with the user initially, but we will take them through at least 2 pages before they complete the process and need to be redirected.  So the first thing we have to decide:  where will we maintain the return URL?  It comes in as a querystring so of course it could be maintained there, but that's a bit of a pain.  Anytime the user submits data to the application, we have to POST it, then we have to reapply it as a querystring for a redirect to we do within the application.  That can be done, but a lot of tracking.  Instead, I'd prefer to just set it once at the beginning and then leave it alone (while having it be available for free when I actually want it).  So the first decision:  we will store it in a cookie.  This is easy to set and retrieve, it's more difficult for the user to casually modify, and there are a couple MVC constructs that can assist us in our task.  

So we know we're putting it in a cookie, how do we get it in there?  Let's start with this interface:

````csharp

  public interface IUrlProvider
    {
        string GetUrl(HttpContextBase context);
    }

````

So given an instance of HttpContextBase, we will return a URL from it.  Okay, simple enough.  Our requirements now lead us to a couple implementations.  First, an implementation over top of the TARGET querystring parameter:

````csharp

	public class SiteMinderTargetQueryStringParameterProvider : IUrlProvider
    {
        private readonly string targetParameterName;
        private readonly IEnumerable<TargetUrlPrefix> parameterPrefixes;
        
        public SiteMinderTargetQueryStringParameterProvider()
            : this("TARGET", new string[] { "-SM-", "$SM$" })
        {
        }

        public SiteMinderTargetQueryStringParameterProvider(string targetParameterName, string[] parameterPrefixes)
        {
            if (string.IsNullOrEmpty("targetParameterName"))
                throw new ArgumentNullException("targetParameterName");

            if (parameterPrefixes == null)
                throw new ArgumentNullException("parameterPrefixes");

            this.targetParameterName = targetParameterName;
            this.parameterPrefixes = parameterPrefixes.Select(a => new TargetUrlPrefix(a));
        }


        public string GetUrl(HttpContextBase context)
        {
            var possibleTarget = context.Request.QueryString[this.targetParameterName];

            if (string.IsNullOrEmpty(possibleTarget))
                return null;

            var urls = this.parameterPrefixes.Select(a => a.AsAbsoluteUri(possibleTarget));

            return urls.DefaultIfEmpty(null).FirstOrDefault(a => a != null);
        }

        public class TargetUrlPrefix
        {
            private readonly string prefix;
            private readonly char smSeparator;

            public TargetUrlPrefix(string prefix)
            {
                this.prefix = prefix;
                this.smSeparator = prefix[0];
            }

            public bool StartsWithPrefix(string possibleTarget)
            {
                return possibleTarget.StartsWith(this.prefix, StringComparison.OrdinalIgnoreCase);
            }

            public bool IsWellFormedAbsoluteUri(string possibleTarget)
            {
                if (!StartsWithPrefix(possibleTarget))
                    return false;

                return Uri.IsWellFormedUriString(Uri.UnescapeDataString(possibleTarget.Replace(this.prefix, string.Empty)), UriKind.Absolute);
            }

            /// <summary>
            /// This method returns an AbsoluteUri which is $SM$ decoded (if it was encoded) per rules at:
            /// http://www.ca.com/us/support/ca-support-online/product-content/knowledgebase-articles/tec555131.aspx
            /// </summary>
            /// <param name="possibleTarget"></param>
            /// <returns>string</returns>
            public string AsAbsoluteUri(string possibleTarget)
            {
                if (!IsWellFormedAbsoluteUri(possibleTarget))
                    return null;

                var absoluteUri = Decode(possibleTarget);

                return new Uri(absoluteUri, UriKind.Absolute).AbsoluteUri;
            }

            /// <summary>
            /// This method decodes $SM$ encoded targets per CA's decode rules found at:
            /// http://www.ca.com/us/support/ca-support-online/product-content/knowledgebase-articles/tec555131.aspx
            /// </summary>
            /// <param name="smEncodedTarget"></param>
            /// <returns>string</returns>
            private string Decode(string smEncodedTarget)
            {
                var sb = new StringBuilder();
                string strippedPrefixUrl;

                strippedPrefixUrl = smEncodedTarget.Substring(4, smEncodedTarget.Length - 4);

                for (int i = 0; i < strippedPrefixUrl.Length; i++)
                {
                    if (strippedPrefixUrl[i] == smSeparator )
                    {
                        sb.Append(strippedPrefixUrl[i + 1]);
                        i++;
                    }
                    else if ( strippedPrefixUrl[i] == '%')
                    {
                        sb.Append(Uri.UnescapeDataString(strippedPrefixUrl.Substring(i, 3)));
                        i = i + 2;
                    }
                    else
                    {
                        sb.Append(strippedPrefixUrl[i]);
                    }

                }

                return sb.ToString();
            }
        }
    }

````

Some of the crazy logic you see in there is because the login agent (SA SiteMinder) will encode the URLs in some rather crazy ways so our provider has to account for that and decode it (and this is beyond typical HTML encoding).   But otherwise, you'll see the code just looks for a querystring parameter called TARGET, assumes it's in the form "-SM-[URI]" or "$SM$[URI]", extracts the URI, and returns it from the provider.  If there is no such parameter or the value of it is not in the expected format, it returns null.  It's true that returning null is a rather unsafe approach, but here we're somewhat mirroring the concept of how MVC value providers operate.

URLs provided by SharePoint will be in the form ?ReturnUrl=/_layouts/authenticate.aspx?Source=/some/page.  The key is the ReturnUrl parameter, not necessarily its value (although that is pretty static to be honest).  So we need a provider that will look for a ReturnUrl querystring parameter and either return what it finds or null if nothing is found:

````csharp

	public class ReturnUrlQueryStringParameterUrlProvider : IUrlProvider
    {
        private readonly string returnUrlParameterName;

        public ReturnUrlQueryStringParameterUrlProvider()
            : this("ReturnUrl")
        {
        }

        public ReturnUrlQueryStringParameterUrlProvider(string returnUrlParameterName)
        {
            if (string.IsNullOrEmpty("returnUrlParameterName"))
                throw new ArgumentNullException("returnUrlParameterName");

            this.returnUrlParameterName = returnUrlParameterName;
        }

        public string GetUrl(HttpContextBase context)
        {
            var returnUrl = context.Request.QueryString[this.returnUrlParameterName];

            if (string.IsNullOrEmpty(returnUrl))
                return null;

            if (!Uri.IsWellFormedUriString(returnUrl, UriKind.RelativeOrAbsolute))
                return null;

            return returnUrl;
        }
    }
	
````

You can see this provider is pretty straightforward, just looks for the parameter, ensures it's a valid URI string, and returns it.  So we're all good here, but there's a couple cases we hvaen't covered.  What if there is no return URL or target parameter specified, what should we do?  And what if there's none specified, but the cookie already has a value (such as after a POST has occurred)?  With no more suspense, we end up with a couple more providers to round things out:

````csharp

	public class ReturnUrlCookieUrlProvider : IUrlProvider
    {
        private readonly string returnUrlCookieName;

        public ReturnUrlCookieUrlProvider(string returnUrlCookieName)
        {
            if (string.IsNullOrEmpty(returnUrlCookieName))
                throw new ArgumentNullException("returnUrlCookieName");
            
            this.returnUrlCookieName = returnUrlCookieName;
        }

        public string GetUrl(HttpContextBase context)
        {
            var returnUrlKey = context.Request.Cookies.AllKeys.FirstOrDefault(a => String.Equals(this.returnUrlCookieName, a, StringComparison.OrdinalIgnoreCase));

            return (returnUrlKey == null) ? null : context.Request.Cookies[returnUrlKey].Value;
        }
    }
	
	public class CompositeUrlProvider : IUrlProvider
    {
        private readonly IList<IUrlProvider> providers;

        public CompositeUrlProvider(params IUrlProvider[] providers)
        {
            if (providers == null || !providers.Any())
                throw new ArgumentNullException("providers");
				
			this.providers = providers.ToList();
        }

        public string GetUrl(HttpContextBase context)
        {
            var urls = providers.Select(a => a.GetUrl(context));

            if (urls.All(a => string.IsNullOrEmpty(a)))
                return null;
            else
                return urls.First(a => !string.IsNullOrEmpty(a));
        }
    }
	
````

So now we have providers that will retrieve URLs from our two querystring parameters and our cookie, and we can roll them together into a single composite provider.  A few things are noticeably absent here:  we do not have a provider that provides a "default" return URL nor have we accounted for any validation.  Both are things we'll need and you could even argue that the first (a default URL provider) would fit in nicely here, but we went a slightly different direction as you'll see. 

<h2>Validation Time</h2>

Our providers give us easy access to the return URL for the user based on the active request, but simply put, the user (or rather their input) cannot be trusted so we need to validate it to decide if we will use it or not.  We first need to make a critical decision:  how will we deal with URLs that could be both relative and absolute?  The SiteMinder agent will always provide absolute URIs, but SharePoint will provide relative ones.  Since we happen to know that the login application will not be deployed on the SharePoint servers, the relative URLs are simply not an option (at least by themselves) so we need to ensure that all return  URLs are in absolute form in addition to being safe to use.   Let's define this interface:

````csharp

	public interface ISafeUrlRule
    {
        bool CanApply(string url);

        Uri GetSafeAbsoluteUrl(string url);
    }

````

An `ISafeUrlRule` provides a way to get a safe absolute URL from the string representation of a URL.  The rule also provides a way to assess if the rule will apply or not should that come in handy.  Here's the most basic rule:

````csharp

	public class SafeAbsoluteUrlRule : ISafeUrlRule
    {
        private readonly IEnumerable<string> safeAuthorities;

        public SafeAbsoluteUrlRule(IEnumerable<string> safeAuthorities)
        {
            this.safeAuthorities = safeAuthorities;
        }

        public bool CanApply(string url)
        {
            if (!Uri.IsWellFormedUriString(url, UriKind.Absolute))
                return false;

            var uri = new Uri(url, UriKind.Absolute);

            return safeAuthorities.Contains(uri.GetLeftPart(UriPartial.Authority));
        }

        public Uri GetSafeAbsoluteUrl(string url)
        {
            if (!CanApply(url))
                return null;

            return new Uri(url, UriKind.Absolute);
        }

        public IEnumerable<string> SafeAuthorities { get { return this.safeAuthorities; } }
    }

````

The `SafeAbsoluteUrlRule` maintains a list of safe authorities (http://www.mysite1.com, http://www.mysite2.com, https://www.mysite1.com) and it will simply compare the authority (including protocol) against that of the url parameter (assuming the url was previously validated as being a well-formed URI); if the url is at a safe authority, the rule simply returns the URL as a Uri.  Otherwise, it returns null.  That's it, pretty simple, and this will work fine for our SiteMinder-provided URLs.  Now let's get a little more exotic:  SharePoint return URLs.  We said already that SharePoint will give us relative URLs, but since we have multiple SharePoint farms (the enterprise farm plus some sub-farms for partner organizations we work with), it's not as simple as using a static base authority.  Really what we want to do is figure out what authority they came from and send them back to the URL there.  Enter `UseHttpRefererAsDefaultAuthoritySafeRelativeUrlRule`.  

````csharp

	public class UseHttpRefererAsDefaultAuthoritySafeRelativeUrlRule : ISafeUrlRule
    {
        private readonly Func<HttpContextBase> httpContext;
        private readonly string defaultAuthority;
       
        public UseHttpRefererAsDefaultAuthoritySafeRelativeUrlRule(string defaultAuthority)
            : this(defaultAuthority, () => new HttpContextWrapper(HttpContext.Current))
        {
        }
      
        public UseHttpRefererAsDefaultAuthoritySafeRelativeUrlRule(string defaultAuthority, Func<HttpContextBase> httpContext)
        {
            this.httpContext = httpContext;
            this.defaultAuthority = defaultAuthority;
        }

        private string GetRefererOrDefault()
        {
            var referer = this.httpContext().Request.Headers["Referer"];
            
            return referer != null ? referer : this.defaultAuthority;
        }

        protected virtual ISafeUrlRule CreateRelativeRule()
        {
            var authority = new Uri(GetRefererOrDefault(), UriKind.Absolute).GetLeftPart(UriPartial.Authority);

            return new SafeRelativeUrlRule(authority);
        }
      
        public bool CanApply(string url)
        {
            return CreateRelativeRule().CanApply(url);
        }

        public Uri GetSafeAbsoluteUrl(string url)
        {
            return CreateRelativeRule().GetSafeAbsoluteUrl(url);
        }
    }
	
	 public class SafeRelativeUrlRule : ISafeUrlRule
    {
        private readonly Uri safeUrlAuthority;
       
        public SafeRelativeUrlRule(string safeUrlAuthority)
        {
            this.safeUrlAuthority = new Uri(safeUrlAuthority, UriKind.Absolute);
        }

        public bool CanApply(string url)
        {
            return (!string.IsNullOrEmpty(url) && Uri.IsWellFormedUriString(url, UriKind.Relative));
        }

        public Uri GetSafeAbsoluteUrl(string url)
        {
            if (!CanApply(url))
                return null;

            return new Uri(this.safeUrlAuthority, url);
        }

        public Uri SafeAuthority { get { return this.safeUrlAuthority; } }
    }

````

No real surprises here.  First, the rule is provided with a way to access the current `HttpContext`.  We provide it as a lambda since Windsor will generate the component for us and the HTTP context is not always in its full-formed state when that occurs.  Plus we may like to register the rule as a singleton.  The rule relies on the value of the "Referer" HTTP header to provide the authority, but the rule checks the authority against its internally held whitelist; if it's not safe, it uses the default authority (basically the SP enterprise farm) instead.  The rule then defaults to the `SafeRelativeUriRule` to actually complete the validation and produce the URI.  

Now that we've got the rules (and we actually have some special one-off rules I'm going to omit), let's raise things up a level to a policy:

````csharp

	public interface ISafeUrlPolicy
    {
        Uri GetSafeAbsoluteUrl(string url);

        bool IsSafe(string url);

        Uri DefaultUrl { get; }
    }
	
	public class SafeUrlPolicy : ISafeUrlPolicy
    {
        private readonly IList<ISafeUrlRule> rules;
        private readonly DefaultUrlRule defaultUrlRule;

        public SafeUrlPolicy(string defaultSafeUrl, params ISafeUrlRule[] rules)
        {
            this.rules = rules.ToList();
            this.defaultUrlRule = new DefaultUrlRule(defaultSafeUrl);
        }

        public bool IsSafe(string url)
        {
            return this.rules.Any(a => a.CanApply(url));
        }
      
        public Uri GetSafeAbsoluteUrl(string url)
        {
            var satisfiedRule = rules.FirstOrDefault(a => a.CanApply(url));

            return ChooseRule(satisfiedRule).GetSafeAbsoluteUrl(url);
        }

        private ISafeUrlRule ChooseRule(ISafeUrlRule selectedRule)
        {
            return selectedRule ?? this.DefaultRule;
        }

        public Uri DefaultUrl { get { return this.defaultUrlRule.DefaultUrl; } }

        public IList<ISafeUrlRule> Rules { get { return this.rules; } }

        public ISafeUrlRule DefaultRule { get { return this.defaultUrlRule; } }
    }

````

The policy is simply a collection of rules plus a special default rule to apply if no other rules work.  There are other ways we arguably should have done this (such as just making it a composite with a default fall-through rule), but we liked how explicit this was.  The policy is what's provided to other components in the application that need to evaluate if a URL is safe per our official policy.  

Since this post is already too long, we'll call this Part 1 and stop here.  But come back for Part 2 as we still have some key items to cover:  how do we link the URL providers and the safe URL rules together and get them to persist the return URL through a cookie?  How will we manage the cookie (when to update it, when to not, etc)?  And ultimately how will we make the return URL available in a frictionless fashion in our application?  All these and more will be explored next time. 

