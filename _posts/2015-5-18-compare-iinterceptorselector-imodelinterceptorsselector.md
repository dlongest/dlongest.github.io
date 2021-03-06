---
layout: post
title: Choose Between IInterceptorSelector and IModelInterceptorsSelector in Castle Windsor
tags: design, castle windsor
description: Confused about which of these types you need to use to vary interceptors at runtime? Those questions now answered.
---
Interceptors in Castle Windsor are one of the easiest and most powerful constructs within the framework.  Interceptors make it almost trivial to implement cross-cutting concerns such as logging or error handling, but really give you a way to share almost any logic across types with almost no ongoing maintenance burden.  And typically this begins with an interceptor that should always be applied to a given component and such a registration is very easy:

````c#

container.Register(Component.For<IMyService>().ImplementedBy<TheService>().Interceptors(typeof(MyInterceptor)));

````

Suppose, though, we want to determine at runtime whether an interceptor should be applied?  For example, what if a configuration setting allows us to toggle the interceptor on and off - how would we handle that?  Well of course the brute force way is:

````c#

var enableInterceptor = GetConfigSettingOnOffState();

if (enableInterceptor)
	container.Register(Component.For<IMyService>().ImplementedBy<TheService>().Interceptors(typeof(MyInterceptor)));
else 
	container.Register(Component.For<IMyService>().ImplementedBy<TheService>());

````

That certainly does the job, but it's ugly.  We're duplicating our registration, which adds to the maintenance burden.  The good news:  there is a better way! The bad news:  there are two similarly-named interfaces, `IInterceptorSelector` and `IModelInterceptorsSelector`: how do we know which one to use? 

For the purpose of this post, let's split our access to the Windsor container into two separate categories.  The first phase we will call registration time and this is the time from when the Windsor container is first constructed until the application is ready to begin handling results.  During the registration period, we are actively modifying the container with new component registrations.  The second phase we will call resolution time and this is from when our application begins handling requests until the application terminates.  In this phase, we are no longer modifying the container, but instead it is servicing our application by resolving components upon request (such as in response to HTTP requests or messages).  Using these two phases makes it easier to differentiate these two interfaces.

The other key thing to remember is how interceptors are implemented:  they are generated by the Castle DynamicProxy library as necessary when the container is resolving a service.  If a component has interceptors applied, the container requests DynamicProxy construct proxy instances that contain the interceptor functionality.  

<h3>Implementing IModelInterceptorsSelector</h3>

Here is a sample `IModelInterceptorsSelector`:

````c#

 public class MyInterceptorSelector : IModelInterceptorsSelector
    {
		private readonly bool interceptorEnabled;
	
		public MyInterceptorSelector()
		{
			this.interceptorEnabled = System.Configuration.ConfigurationManager.AppSettings["interceptorSetting"] == "yes";
		}
		
        public bool HasInterceptors(ComponentModel model)
        {     
			return model.Implementation == typeof(IMyService) && this.interceptorEnabled;		
        }

        public Castle.Core.InterceptorReference[] SelectInterceptors(ComponentModel model, 
                                                                    InterceptorReference[] interceptors)
        {
			return interceptors.Concat(new InterceptorReference(typeof(MyInterceptor)));
        }
    }
	
	/// Add interceptor selector into ProxyFactory on the container
	container.Kernel.ProxyFactory.AddInterceptorSelector(new MyInterceptorSelector());
	
````

So what's going on here?  First, `HasInterceptors` defines whether this selector wants to apply interceptors to the component model. Windsor will call this method for a given component model to decide if it should give the selector a chance to provide an interceptor.  If it returns true, Windsor will later call SelectInterceptors.  The collection of interceptors is all those interceptors that have previously been applied to the component model.  The selector *must* return a collection of interceptors that should apply to the model.  At a minimum, that means returning the interceptors argument as-is.  However, typically the selector will add a new interceptor into the collection and return the modified one as shown.  Note that `InterceptorReference` is the argument collection type and we instruct the constructed reference as to our interceptor type.  In our case, `HasInterceptors` is based simply on whether the interceptor should be enabled and that the component model is for the type we care about (`IMyService` in this case).  This is an important point:  once we add the selector to the `ProxyFactory`, Windsor will call our selector for all new component models to give us a chance to specify if an interceptor should be applied to it.  In `SelectInterceptors`, we simply append an `InterceptorReference` to our interceptor to the collection and return it. 

From this, would we say this selector is registration-time or resolution-time?  It's a registration-time extension point and we can infer this from both its use of the component model as an argument plus the fact that it deals in `InterceptorReference` and not `IInterceptor`.  Bottom-line:  `IModelInterceptorsSelector` allows us to stipulate the default set of interceptors that should apply to a given component.  We can add to this list, remove items from the list, or reorder them.  

<h3>Implementing IInterceptorSelector</h3>

Here's an `IInterceptorSelector` that achieves the same *functional* end as `IModelInterceptorsSelector`, but in a completely different way:

````c#

 public class MyInterceptorSelector : IInterceptorSelector
    {
        private readonly bool enableInterceptor;

        public MyInterceptorSelector()
        {
            this.enableInterceptor = System.Configuration.ConfigurationManager.AppSettings["enableMyInterceptor"] == "yes";
        }
        public IInterceptor[] SelectInterceptors(Type type, MethodInfo method, IInterceptor[] interceptors)
        {
           if (!enableInterceptor)
		   {
				return interceptors.Where(a => !a.GetType() == typeof(IMyService));
		   }
		   
		   return interceptors;
        }
    }

````

Now we've got one method to implement that takes a Type, a MethodInfo, and a collection of IInterceptors.  The lattermost of these are "live" instances - they have been fully constructed by the container, but not yet executed.  This immediately tells us that `IInterceptorSelector` is a resolution-time construct.  Unlike `IModelInterceptorsSelector`, we no longer have the ability to add new interceptors to the collection as we do not have the means to construct an interceptor (other than manually, which if we're using Windsor, is sort of defeating the purpose).  What we *can* do is filter or reorder the collection of interceptors.  In this case, we simply assess if the interceptor should be enabled and if not, we filter it out of the list of interceptors.  Unsurprisingly, this is a resolution-time component, which we can determine based largely on the fact that we receive a collection of already-constructed interceptors.  The other big point:  this allows us to determine if an interceptor should only apply to *some* methods of a type, but not necessarily all. 

<h3>So How Do I Choose?</h3>

If you want an interceptor to apply to all methods on a type, use an `IModelInterceptorsSelector` to specify that it is one of the default interceptors for the type.  This is most likely to keep the proxy factory performance at a good level since proxies can be cached and reused since our logic applies at the `InterceptorReference` level prior to live instances being constructed.  

If you want an interceptor to apply to a given type, but *only* to certain methods, `IInterceptorSelector` is your only choice between these two components since the other is at the model level, not the method level.  Again, this could have performance impacts on the ability of proxies to be cached so be careful overusing this approach.  If you find yourself doing method selection quite heavily, it might make more sense to have the actual `IInterceptor` instance perform its own check on the method (based on `IInvocation` passed to it).  This muddies things a bit, but the performance gain may be substantial enough to justify it, particularly if you have a lot of `IInterceptorSelector`s performing per-method filtering.  


