# Error Categories

## Context

The Problem: We're looking for a handful of categories to classify every error possible.

My team was in the not-so unique position of having to rewrite our service protocol, migrating it from an OData protocol to a gRPC protocol. An additional requirement of this rewrite was that we also write a client that can provide an abstraction that works over both of these protocols. The protocol we migrated was for a service that is a fairly straightforward generic document storage solution, so the "happy path" scenarios (those which return no error) were simple to write abstractions over.

However our team got caught in a bit of debate when considering how we return errors. Due to many factors that are no longer under our control, our services returns a large number of unique error codes-- too many to expect consumers to know each one. As a result, the vast majority of users don't even look at the error code. Instead they will evaluate HTTP Status Code when deciding what action to take with an error response.

This presented a difficulty for creating our protocol abstraction. gRPC has a separate set of error codes than HTTP provides, and they don't all map neatly to each other. We *could* create a manifest of error code to HTTP Status Codes moving forward and expose a fake HTTP Status Code to the user for reach request, but we worried that (given the size of our organization) the someone may make a new error code and forget to update the clients to generate the mapping. We *could* send over the HTTP Status Code as part of the gRPC payload, but now our service has decided to forever couple itself to HTTP concepts even if we wanted to use a different base protocol.

One conclusion we came to was that sending a category would be a good way to ensure forward compatibility, but we as a team were unable to agree on any categories we invented. Given our time constraints to get this shipped, we never went with that decision. Though our product already shipped, I continued to wonder if there was a better set of error categories than what HTTP presents. One evening I got the idea to categorize the errors by component of directive. Here's what that idea looks like.

> Note!: These are not intended to replace HTTP Status Codes. Those have specific uses and relationships with web-browsers and web-servers. These are intended to replace the use of HTTP Status Codes as error categories in an application protocol. These would also work well as an addition to existing protocols which do not currently categorize their error codes, and have to have too many error codes as is reasonable for a developer to remember.

## What is a Directive?

Every directive is can be decomposed of the following parts:

- The actor: the person or thing who wants the directive done.
- The subject: the person or thing who the command is being given to.
- The language: the set of shared symbols both the actor and the subject understand.
- The specifications: the action the actor wants the subject to take.
- The context: the way the world is at the time the command is given.

While this breakdown could be applied to any directive, the goal of this categorization effort is to make error handling easier and error messages clearer. We'll start with an example in the context of a web request, consider the following scenario:

> A user is on a website that allows them to order custom pies. The user is at a screen which allows them to specify how many cherries they want on top of their pie. The screen also provides an "order" button which starts the process that will eventually deliver their pie. The website is written so that when the "order" button is clicked, the data from the form is transformed into a web request that gets sent to a server for processing.

Because the proposed breakdown is so abstract, you could identify many levels of this abstraction within this scenario. For example, one application of the abstraction that we're **not** interested could be: the actor would be the user ordering the pie, and the subject would be the bakery. Instead, we will focus on web request submitted when the "order" button is pressed.

The actor for our scenario will be the web site which has crafted the payload and is attempting to submit the order to the backend service. The subject is the entire backend service, which is receiving and processing the request. The language is the rules of the RESTful API which was implemented (and hopefully documented) by the service developers. The specifications are the details that the user wants one pie, and that the user wants 27 cherries on it. The context is too large to describe, but it's important to note that it include things like the fact that the backend service keeps track of how many cherries are available, and at the time of order it's only 26.

Using our new framework, we can identify all things that can go wrong as belonging to those components. Consider the following examples:

- The backend service is down entirely: an error of subject
- The payload was formed with the amount of cherries in the wrong field name: an error of language
- The backend service only allows users to order less than 25 cherries per order: an error of specifications
- The backend service only has availability for 26 cherries: an error of context

The benefit of using this abstraction is that we can rely on people's intuitive understanding of a directive to provide better communication to the user what went wrong, instead of trying to rely on computing concepts or giving an overly generic error. We can also rely on developers' intuitive understand of a directive to generalize debugging steps of an error in each component, and acknowledge the information a servicer could provide to help debugging (hopefully to facilitate semi-automated or at least quicker investigations.)

## Refining for Web Services

If we were to translate these into a set of standardized error category codes, we could expect them to look like this: `BadActor`, `BadSubject`, `BadLanguage`, `BadSpecifications` and `BadContext`. But in the context of a web request there are certain scenario constants that we can identify and optimize these codes around.

The `BadActor` category code is to be returned when the actor in the situation is purposefully acting maliciously. This category code is likely to be used as part of a service protection mechanism to indicate that no matter what the user/client is doing they will not get service. A service without protection will not need to reference this category.

The `BadSubject` category code would be returned when the service itself, or some essential component is acting unexpectedly. Given that the subject is a conceptual constant (always "the backend"), we will rename this to `BadService`.  This most closely maps HTTP's with bad gateway errors and service availability errors.

The `BadLanguage` category code would be returned when the payload is shaped unexpectedly. However given the common existing Computer Science conceptualization of a "language", we will rename this to `BadFormat`. This error most closely maps with HTTP's bad request error. An interesting note about this category is that it is theoretically possible to eliminate. You could either write your language to be robust enough to accept any possible input, or exist within a framework where the invalid inputs are nearly impossible to provide. Check out the section on using this framework within a compiled and single-process context for more information.

The `BadSpecifications` category code would be returned when the data described by the payload is in an unexpected range. While this category is useful, it is frequently the case that the specifications are broken into two common components: the object and the arguments. The term "object" is being used here in a grammatical sense: it refers to the entity which receives the action of the request (for example the "document" part in a "get document" call). Therefore we will split this error category code into two category codes: `BadObject` and `BadArgument`. `BadObject` is to be returned when the object of the directive is acting unexpectedly (either in an entirely invalid state or does not exist.) This most closely maps with HTTP's not found error. In addition, many services use the not found error to obfuscate access denied errors-- pretending that if the actor does not have access to the object, it simply does not exist. In these scenarios this is still the error category code is to be returned. `BadArgument` is to be returned when the other values provided in the request are unexpected for the properties they are applied to. This again most closely maps with HTTP's bad request error. The value added by separating the bad request status code into two error categories is that you can easily express to the user or developer two separate problems: the value provided is not understood by the server; and the value is not allowed by some application logic. This makes parsing logs easier as one category indicates they have a bug, while the other indicates user is doing something unallowed. And surfacing these categories to the user (not directly) allows the developer to communicate in one case that the client they are using is broken, and in another case when the specifics of what they want is not allowed.

The `BadContext` category should be returned when some precondition external to the details of the request was not met. This category implies that entities outside of this request need to be changed before moving forward. I wouldn't argue that this maps with HTTP's Too Many Requests error code, but rather that the Too Many Requests error code is an example of such a case of bad context. The user is expected to wait for an allotment of requests. This is useful to describe to the user and developer that nothing they can change about their current request would be helpful in getting them to their intent.

The next section is intended as a quick guide to refer back to when implementing these error codes on your service. It also provides some implementation suggestions to help users, developers, and service owners.

## Quick Guide for Network Requests

* **BadService** - If the service was not in a state to perform the action.
  * *Suggested Default End User Message*: Something went wrong getting your request to the server. Try again later.
  * *Suggested Debug Steps*: Verify the servicer and all dependencies are running.
  * *Suggested Servicer Facilitation Features*: Offer an API which reports the health of itself and all dependent components.
  * *Retryability*: Automatically only if the baseline service has a reputation for recovering quickly. Manually only if the project is being maintained.
  * *HTTP Mappings*: InternalServerError, ServiceNotAvailable
* **BadFormat** - If the call was not formatted in a way that was understandable by the servicer.
  * *Suggested Default End User Message*: Something went wrong making your request. Try again on another device, or after updating this one.
  * *Suggested Debug Steps*:
    * Check additional details on the error response to identify command expectation.
    * Identify the shape of the request.
    * Review servicer documentation on expected shapes, and compare to identified shape.
  * *Suggested Servicer Facilitation Features*:
    * Provide clear documentation on request shapes.
    * Provide a field in the error response which indicates which component of the request was unreadable.
  * *Retryability*: Automatically never. Manually if you have access to different versions of your client library that you can try again on.
  * *HTTP Mappings*: BadRequest, Unauthorized
* **BadObject** - If the object does not exist, or the object not accessible to the caller.
  * *Suggested Default End User Message*: The `Entity Type` you are looking for does not exist, or you do not have access to it.
  * *Suggested Debug Steps*:
    * Obtain proof the create action has been made on this object.
    * Verify the current identity of the actor.
  * *Suggested Servicer Facilitation Features*: Note: This error is often very difficult for users to debug, given the purposeful obfuscation methods of common security practices.
    * Offer an API which reports details known about the identity of the actor making the call.
    * Implement clear internal logging on whether this object exists, at what time the object has been made and when this check was performed, or whether it was purposefully withheld for security. If it was withheld for security, clear logging should exist around any details that would indicate what objects an actor has access to.
  * *Retryability*: Automatically never. Manually only after the user has done something to change their access.
  * *HTTP Mappings*: Forbidden, NotFound
* **BadArgument** - If the argument is not in the expected range of values for the current state.
  * *Suggested Default End User Message*: `Property Name` cannot be `Property Value`
  * *Suggested Debug Steps*:
    * Check additional details on the error response to identify valid values.
    * Start changing what properties *can* be changed.
  * *Suggested Servicer Facilitation Features*:
    * Offer clear identification of the specific property which was not in a valid range.
    * Offer clear identification of the valid range for the given property.
  * *Retryability*: Automatically never. Manually only after the property has changed.
  * *HTTP Mappings*: BadRequest
* **BadContext** - If some external precondition unrelated to the object was not met.
  * *Suggested Default End User Message*: `Entity Name` cannot currently do that.
  * *Suggested Debug Steps*:
    * Check additional details on the error response to identify the unmet precondition.
    * Start changing what *can* be changed, starting from the closest entities/properties from the object being modified.
  * *Suggested Servicer Facilitation Features*: Offer clear identification of the specific precondition that was not met.
  * *Retryability*: Automatically never. Manually only after the user has changed the appropriate preconditions.
  * *HTTP Mappings*: Too Many Requests
* **BadActor** - If the user is identified as a malicious actor.
  * *Suggested Default End User Message*: Please contact support.
  * *Suggested Debug Steps*: Contact the service owners.
  * *Retryability*: Never.

## Refining for Compiled Single Process Requests

The purpose of a compiled runtime is to eliminate entire categories of errors that may occur. In this context it is also difficult to choose the right abstraction: each line is technically a directive, where the actor is the program, and the subject is the processor. No abstraction comes without it's cognitive overhead, so these category codes should only be chosen when your function has a large amount of error return conditions. If you're going to use these in a compiled environment, here are some suggested modifications.

`BadActor` should likely always be an exception or some other short circuit. Your program should take a defensive position and try to offload the requests as quickly as possible.

`BadFormat` is nearly entirely eliminated. But cosmic rays can flip bits to make previously valid directives invalid. So if you can somehow detect this, I recommend returning this.

This isn't a modification, just an interesting note: `BadSubject` is the implicit category code of most Null Reference Exceptions in object oriented languages. Modern languages can also nearly entirely eliminate this category.

## Bonus: Directive Breakdown on Human Communication

I stated above that it works for any general directives. I thought it'd be fun to do the exercise of applying this abstraction for human communication. Consider the following directive:

> A parent and child are sitting at a dinner table with plates containing broccoli. The parent directs the child, "Eat your broccoli."

The actor is the parent. The subject is the child. The language is English. The specifications are to consume the broccoli which belongs to the child. The context in this scenario could be an entire universe worth of information. But if, say for example the broccoli was too hot to safely eat, this could be considered a misunderstanding of context by the parent, and therefore the child has a right to shout, "BAD CONTEXT."

