---
layout: post
title: Extending the MVC Default Model Binder
tags: MVC, design
---
In the [last post]({% post_url 2014-12-14-writing-a-custom-mvc-model-validator %}), I talked about authoring custom MVC model validators that can be plugged into MVC's model validation pipeline.  In this post, I will go into some details on how we used a custom model binder to execute some logic after default model validation, but prior to MVC's model validation being executed.  

In the login application we were writing, there were two cases when we wanted to update the model prior to validation (so that validation would apply), but I didn't want to write a separate per-type model binder.  In MVC, you can define the application-wide model binder in the application root by doing:

````csharp

ModelBinders.Binders.DefaultBinder = new MyCustomModelBinder();

````

This binder will completely replace the default binder although generally it's easiest to inherit from the `DefaultModelBinder` and just extend the necessary method.  But figuring out exactly what to extend and what to reuse isn't necessarily obvious.  

In our case, there were a few things we needed to do:
- We needed to pass values between web requests, but in some cases the values were sensitive so we encrypted them as hidden fields, but needed to decrypt them automatically during model binding
- The vendor stack that actually provides the backing store will provide some user-specific data in the HTTP header on certain views and we wanted to automatically bind those

In both cases, it would be possible to extend one of the MVC components to do these jobs, but a couple things bothered me:
- Each time we came up with additional model binding that needed to occur, we would have to implement a new model binder and figure out how to incorporate previously logic or violate Open-Closed and keep making changes to our binder.
- We could write a HTTP Header Value Provider, but it was only needed a fraction of times for a couple specific fields and the property on the view model we would populate had a different name so would have to figure out some alias system
- It is possible to write per-type model binders and that might have worked for the cases where we were populating header values into the view model (since there were really 3 use-case specific ones of these), but our decryption can theoretically apply to fields on any view model and I didn't want to have to define a per-type binder when the goal of encryption was to simply annotate properties and have it work. 

So it occurred to us that what we wanted to have happen was:
- Use the `DefaultModelBinder` up to the point when it runs model validation
- At this point, run components against the completely bound model that would update the model as they saw fit (again, similar to model binding, hence the timing)
- Once all those components completed, have the `DefaultModelBinder` run its custom validation logic as it normally would. 

This brings me to the first key in the process:  what method in the `DefaultModelBinder` needs to be overridden to give us access to a completely bound but not yet validated view model?  The answer:  OnModelUpdated.  The below snippet shows the implementation of the OnModelUpdated method on DefaultModelBinder pulled from the MVC source code on CodePlex.  This is the only method that uses the model validators by way of using the application-wide static ModelValidator instance to get the validators to iterate through.  

````csharp
protected virtual void OnModelUpdated(ControllerContext controllerContext, ModelBindingContext bindingContext)
        {
            Dictionary<string, bool> startedValid = new Dictionary<string, bool>(StringComparer.OrdinalIgnoreCase);

            foreach (ModelValidationResult validationResult in ModelValidator.GetModelValidator(bindingContext.ModelMetadata, controllerContext).Validate(null))
            {
                string subPropertyName = CreateSubPropertyName(bindingContext.ModelName, validationResult.MemberName);

                if (!startedValid.ContainsKey(subPropertyName))
                {
                    startedValid[subPropertyName] = bindingContext.ModelState.IsValidField(subPropertyName);
                }

                if (startedValid[subPropertyName])
                {
                    bindingContext.ModelState.AddModelError(subPropertyName, validationResult.Message);
                }
            }
        }
		
````

So at this point, we know one thing we need to do: implement our own model binder that inherits from `DefaultModelBinder`, overrides OnModelUpdated to hook in our custom logic, and ultimately call base.OnModelUpdated to run the default validation.  But before we implement that, we need to think about what our new update components will look like. 

Conceptually, we're going to pass the model from the model binder (and the model can be of any type) to a component and that component is going to potentially modify properties on it directly based on its rules.  So let's define an interface `IModelUpdate` which does that, with the addition of ControllerContext:

````csharp

	public interface IModelUpdate
    {
        void Update(object model, ControllerContext controllerContext);
    }
````

As mentioned, the model updates we'd like to apply are:
- Decrypt specific properties based on annotations
- Bind values from the HTTP request

These become several implementations of `IModelUpdate` that are then plugged into our custom model binder, which looks like:

````csharp

public class PreValidationModelUpdatingModelBinder : DefaultModelBinder
    {
        private readonly IList<IModelUpdate> modelUpdates;

        public PreValidationModelUpdatingModelBinder(params IModelUpdate[] modelUpdates)
        {
            if (modelUpdates == null)
                this.modelUpdates = new List<IModelUpdate>();
            else
                this.modelUpdates = modelUpdates.ToList();
        }      

        protected override void OnModelUpdated(ControllerContext controllerContext, ModelBindingContext bindingContext)
        {
            this.modelUpdates.ToList().ForEach(a => a.Update(bindingContext.Model, controllerContext));           
                
            base.OnModelUpdated(controllerContext, bindingContext);            
        }       
    }    
	
````

As you can see, our model binder simply takes a collection of IModelUpdate components, overrides OnModelUpdated, calls each component, and then calls base.OnModelUpdated to have the default model binder run the model validations.  Each model updater is responsible for determining
if it actually cares to perform an update on the model; if not, it simply does nothing to it.  I've omitted the implementations of the `IModelUpdate` components as they're pretty application specific, but they are in my [Github](https://gist.github.com/dlongest/387d2ce35fb6f45688b8) if you'd like to see them.  

This concludes a look at how we extended MVC to leverage its existing binding and validation functionality while letting us extend it in some pretty unique ways. 



