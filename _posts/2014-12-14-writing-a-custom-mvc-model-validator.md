---
layout: post
title: Writing a Custom MVC Model Validator
tags: MVC, design
---
The power of ASP.NET MVC is how extensible the framework is, but with this comes a challenge:  in certain cases as a developer you know (or have a feeling) that there is an extension point, but how do you find and use the right one?  In a series of posts, I will explore the myriad extension points in MVC from the perspective of a project I've been working on for a few months.  Like most development, every situation has an almost limitless number of options for accomplishing it, but with regards to extending MVC in my experience there is often one "best" choice, the key is being aware of it.  

To establish some context, here's the project: a login application.  There are thousands (or millions) of these available so why would we be writing our own?  I work for a not-for-profit company and we made the decision to license a popular identity access management (IAM) platform from a vendor and partner with a second company to implement and support it for us.  Initially, there had been no development work planned on our side at all: the implementer would take care of all of it with an out-of-the-box login application configured against various cloud services (provided by the solution vendor, but running in the implementer's cloud).  However, the OOTB application was not responsive, which was a big deal for us so the decision was made for us to write our own.  Conceptually, this seemed straightforward enough: the login application would (in theory) simply make calls to either a REST service (for logging in) or a SOAP service (for most other identity functions, password resets and the like).  However, over time our custom login application began to accrue more and more functionality as the implementer struggled to support our requirements within the vendor solution.  It is this difficulty supporting some requirements that led us to extending the framework in various ways. 

There are plenty of other posts and articles out on the web that describe the order of certain actions in the framework and I won't go into those in detail, but the key is this order:

* Model binding (which includes model validation)
* Filters (authorization, action, result)

Model binding is the set of components that populate a model (which is often a view model, but can be any type which is the parameter for a controller action), using values pulled from the value provider.  A value provider is an abstraction over the HTTP request itself plus whatever custom value providers have been defined.  Under the hood there are several different default providers (one for querystring values, one for posted data, etc), but they are gathered together into a composite pattern (so if you look at the ControllerContext.Controller.ValueProvider, it is a single instance).  As the binding progresses, the model binder fetches model validators out of the framework's static ````ModelValidatorProviders```` collection and executes each validator.  An individual model validator looks at some part of the model that it cares about and generates a `ModelValidationResult` containing error text if the validation fails that the model binder then writes into the model state error. 

So here's the situation: like most enterprises, we have specific rules governing a user's password, but the vendor IAM platform simply could not be configured to support our custom rules in concert with disabling certain out of the box rules we didn't want to use (yes, very bizarre, I agree).  What are our custom rules? 

* Can't use certain banned phrases in the password
* Can't use your first name or last name anywhere in the password
* Password must be between 8 and 30 characters long
* Password must have a lowercase character, an uppercase character, and a number
* Password cannot have certain special characters (or said another way, we only support a limited numuber of non-alphanumeric characters)

A few of the rules are supported trivially with model annotations, such as the length (using ````StringLength````) and acceptable alphanumeric characters (using custom RegEx attributes if nothing else).  However, the restrictions on first name and last name and banned phrases are much more difficult, especially: where do the first name and last name come from?  Where do the banned phrases come from?  An attribute could be written to do this, but it would almost definitely not be unit testable. 

It feels in this case like model validation is still the right mechanism to use: it's built into the framework and will run for us automatically so let's start by writing a model validator to validate the password.  

````csharp

 public class PasswordValidator : ModelValidator
    {
                
        public PasswordValidator(IAllowPassword allowPassword, ModelMetadata metadata,
                                 ControllerContext context)
            : base(metadata, context)
        {
            this.PasswordProperty = metadata.ModelType.GetProperties().First(a => Attribute.IsDefined(a, typeof(PasswordFormatAttribute)));
            this.Password = this.PasswordProperty.GetValue(metadata.Model, null) as string;
            this.AllowPasswordPolicy = allowPassword;
        }

        public PropertyInfo PasswordProperty { get; private set; }

        public string Password { get; private set; }

        public IAllowPassword AllowPasswordPolicy { get; private set; }
        

        public override IEnumerable<ModelValidationResult> Validate(object container)
        {
            try
            {
                var notAllowedReasons = this.AllowPasswordPolicy.IsAllowed(this.Password);

                return notAllowedReasons.Select(a => CreateResult(a));

            }
            catch (Exception ex)
            {
                return Enumerable.Empty<ModelValidationResult>();
            }
        }    


        protected ModelValidationResult CreateResult(string message)
        {
            return new ModelValidationResult
            {
                MemberName = "Password",
                Message = message
            };
        }
    }   
	
````

The ````PasswordValidator```` relies on a collection of ````IAllowPassword```` rules to decide if a given password is acceptable or not.  The validator is created by a custom instance of ````ModelValidatorProvider```` which in this case looks up the banned phrsaes from the web.config and then instantiates the other IAllowPassword instances:

````csharp
public class WebConfigPasswordComplexityModelValidatorProvider : ModelValidatorProvider
    {
        private readonly IEnumerable<string> bannedPhrases;        
      
        public WebConfigPasswordComplexityModelValidatorProvider()
        {
            var config = ConfigurationManager.GetSection("passwordComplexityRules");
            
            if (config==null)
            {
                this.bannedPhrases = new List<string>();
            }
            else
            {
                var passwordConfig = (PasswordComplexityRulesConfigurationSection)config;

                this.bannedPhrases = passwordConfig.BannedPhrases
                                       .Cast<PasswordComplexityBannedPhraseConfigurationElement>()
                                       .Select(a => a.Phrase);
            }           
        }

        public override IEnumerable<ModelValidator> GetValidators(ModelMetadata metadata, ControllerContext context)
        {              
            if (!typeof(AutomaticReturnViewModel).IsAssignableFrom(metadata.ModelType))
                return Enumerable.Empty<ModelValidator>();
                        
            if (!metadata.ModelType.GetProperties().Any(a => Attribute.IsDefined(a, typeof(PasswordFormatAttribute))))
                return Enumerable.Empty<ModelValidator>();
            
            return new List<ModelValidator>
            {
                new PasswordValidator(new CompositeAllowPassword(
                                        new CannotUseInPasswordAttributeValuesNotAllowed(metadata.Model),
                                        new PasswordWithBannedPhrasesNotAllowed(this.bannedPhrases)), metadata, context)
            };
        }
    }
````

Our custom ModelValidatorProvider is hooked into the framework in the Global.asax:

````csharp

 ModelValidatorProviders.Providers.Add(new WebConfigPasswordComplexityModelValidatorProvider());

````

Now for the actual ````IAllowPassword```` implementations.

First, ````PasswordWithBannedPhrasesNotAllowed```` takes a collection of banned phrases (simple strings) and just verifies if the possiblePassword contains any of them.  

````csharp

public class PasswordWithBannedPhrasesNotAllowed : IAllowPassword
    {
        public PasswordWithBannedPhrasesNotAllowed(IEnumerable<string> bannedPhrases)
            : this(bannedPhrases, ControllerResources.Password_Phrase_NotAllowed)
        {
        }

        public PasswordWithBannedPhrasesNotAllowed(IEnumerable<string> bannedPhrases, string errorMessage)
        {
            this.BannedPhrases = bannedPhrases ?? new List<string>();
            this.ErrorMessage = errorMessage;
        }

        public IEnumerable<string> BannedPhrases { get; private set; }

        public IEnumerable<string> IsAllowed(string possiblePassword)
        {
            if (this.BannedPhrases.Any(a => possiblePassword.ContainsIgnoreCase(a)))
                return new List<string> { this.ErrorMessage };

            return Enumerable.Empty<string>();
        }

        public string ErrorMessage { get; set; }
    }
````

Next is the more interesting rule that a password cannot contain a first name or last name.  The challenging part of this instance is: how does it know where to find the first name and last name?  Does it have to look it up or is the field available to it somewhere (such as on the model)?  Can it work in a consistent way across any view model that has a password (which, for us, is ChangePasswordViewModel, ResetPasswordViewModel, and CreateAccountViewModel, as all of them allow a user to input a new password)?  

Here's one approach.  First, let's say that the view model needs to have the first name and last name that should be used for validation.  In some cases, the user is entering the first name and last name when they enter a new password so it's a natural fit (e.g. creating an account).  In other cases, first name and last name aren't entered by the user so we must figure out some way to populate them.  That may seem like a hardship (and a later post will outline a way we did that), but think about the component we're writing currently:  a model validator.  The validator does not (and should not) care how a view model is bound: its only precondition is the view model has been completely bound and is ready to be validated.  So we will temporarily shelve the concerns about how we'll put first name and last name into the model and just assume they can be put in there for us by some model binder.  

That leaves only one final issue:  how should our rule know that a given field is a first name or last name field?  One way is to provide it with property names (or property names mapped to a view model type) and that would work fine.  It would be implicit though while most model binding and validation is driven explicitly via annotations (of course, the banned phrase validator above is implicit).  Let's define a ````CannotUseInPasswordAttribute```` (with no behavior) that can be placed on any property on the view model that indicates it cannot be part of the password, then have our rule simply find those properties with that annotation and voila, it knows what fields are ineligible.  Each particular annotation will include the error message that should be written into the model state error (so the message for first name and last name could be different).  

````csharp

 public class CannotUseInPasswordAttribute : Attribute 
    {
        private string errorMessage;

        public string ErrorMessage
        {
            get
            {
                // elided - get from message resource type and name
            }
        }

        public string ErrorMessageResourceName { get; set; }

        public Type ErrorMessageResourceType { get; set; }
    }


 public class CannotUseInPasswordAttributeValuesNotAllowed : IAllowPassword
    {
        public CannotUseInPasswordAttributeValuesNotAllowed(object viewModel)
        {
            this.ViewModel = viewModel;
        }

        public object ViewModel { get; private set; }

        public IEnumerable<string> IsAllowed(string possiblePassword)
        {
            var errorPhrases = GetErrorPhrases();

            var errors = errorPhrases.Select(a => a.IsBanned(possiblePassword)).Where(a => !string.IsNullOrEmpty(a));

            return errors;
        }

        private IEnumerable<ErrorPhrase> GetErrorPhrases()
        {
            var viewModelProperties = this.ViewModel
                                          .GetType()
                                          .GetProperties();

            var cannotUseProperties = viewModelProperties.Where(a => Attribute.IsDefined(a, typeof(CannotUseInPasswordAttribute)));

            var errorPhrases = cannotUseProperties.Select(a => new
            {
                PropertyValue = a.GetValue(this.ViewModel, null) as string,
                ErrorMessage = (a.GetCustomAttributes(typeof(CannotUseInPasswordAttribute), false).First() as CannotUseInPasswordAttribute).ErrorMessage
            })
                                                  .Where(a => a.PropertyValue != null)
                                                  .Select(a => new ErrorPhrase
                                                  {
                                                      BannedPhrase = a.PropertyValue,
                                                      ErrorMessage = a.ErrorMessage
                                                  });

            return errorPhrases;

        }

        private class ErrorPhrase
        {
            public string BannedPhrase { get; set; }
            public string ErrorMessage { get; set; }

            public string IsBanned(string possiblePassword)
            {
                if (possiblePassword.ContainsIgnoreCase(this.BannedPhrase))
                    return this.ErrorMessage;

                return null;
            }
        }
    }
````

At this point, we can update our view models to have the ````CannotUseInPasswordAttribute```` attribute (in our case, there's 4 of them at this point) and everything will work (assuming first name and last name get populated, which we'll deal with separately). 

I went straight to showing the final code, but it developed over time from unit tests.  Below is one such test (using xUnit data theories)

````csharp
[Theory]
        [InlineData("$ome1234phrase")]
        [InlineData("phrase$something")]
        [InlineData("Containsphraseend")]
        [InlineData("differentPHRASEcase")]
        public void ReportsError_WhenPasswordContainsValue_FromOtherAnnotatedProperty(string possiblePassword)
        {
            var viewModel = new SingleCannotUsePropertyViewModel { CannotUse="phrase" };

            var sut = new CannotUseInPasswordAttributeValuesNotAllowed(viewModel);

            var allowed = sut.IsAllowed(possiblePassword);

            Assert.True(allowed.Count() == 1);
            Assert.Equal(ErrorMessages.CannotUse, allowed.First());            
        }
````

This concludes how we used a custom model validator to ensure a password doesn't contain certain proscribed text.  In the next post, we will explore how we extended model binding to actually populate the necessary first and last name properties on the view models.



 