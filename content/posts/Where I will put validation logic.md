---
title: "Where I will put validation logic"
date: 2021-01-01T22:40:33+01:00
draft: false
tags : ["validation"]
categories : ["DDD"]
---
Sometimes we are confused, where to put validation logic. Somebody refers to write it on the application layer (controller/ command handler) and somebody refers to write it in the domain layer if it is business validation. Sometimes we mix data validation with business validation. Here I will try to clear different types of validation and later I will try to describe multiple options for implementation.

### Data Validation
Data validation is the process of verifying whether the value of a data item comes from the given set of acceptable values or not. 
We should validate data when it is coming from external sources. Consider we have a user management system there we would like to create a user. While we are creating a user, the system should validate input to confirm that the coming information is correct. We can consider the following example for data validation.
-	User first and second name should not be empty
-	Email address should be a valid email address and it should not be empty

We can do data validation while receiving the request or we can do data validation from the command handler before invoking the domain action. But according to [Fail-fast principal](https://en.wikipedia.org/wiki/Fail-fast) it is good if we do data validation as early as possible. If we are developing REST API for our system, then it will be good to perform data validation from the controller.

#### Possible ways to implement Data Validation
#### Data validation using Data Annotation
This is the simplest way of doing data validation. This is something like following.
``` cs
 public class User
 {
     public Guid Id { get; set; }
     [Required]
     public string FirstName { get; set; }
     [Required]
     [MaxLength(50)]
     public string LastName { get; set; }
     [Required]
     public string EmailAddress { get; set; }
 }
```
This approach has some drawback 
-	We are mixing validation logic with the model.
-	If we want to use same model for two different operations with two different validation criteria, then we cannot achieve it using data annotation. For example, Id is required when you are going to perform update operation but is optional when you want to perform save.

#### Define validation logic separately (Deferred validation)
This approach provides us the flexibility to define validation logic separately. So business/crud model is separated from data validation, we can call it deferred validation. So our data validator looks like the following. If we want we can use [fluent validation](https://fluentvalidation.net).
```cs
public class UserValidator : AbstractValidator<User>
{
    public UserValidator()
    {
        RuleFor(p => p.FirstName).NotEmpty();
        RuleFor(p => p.LastName).NotEmpty().MaximumLength(50);
        RuleFor(p => p.EmailAddress).NotEmpty().EmailAddress();
    }
}
```
Above approaches are good for both cases when it is CRUD or CQRS. We can perform validation from the controller or from the handler. As already said early failure is good we can invoke it from the controller. 

> Fluent validation has extension that intercept the request and invoke validator before coming to controller action [Details](https://docs.fluentvalidation.net/en/latest/aspnet.html). So we dont need to invoke it manually.
```cs
services.AddMvc().AddFluentValidation();
OR
services.AddControllers().AddFluentValidation();
```
### Invariant or Business rule validation
Business rule validation is totally different concept than data validation. Business rule validation is fully business oriented. Based on business rule/ invariant some action will be performed or not. Business rules are enforced while creating the object or changing its state. 

> business invariants — the rules to which the software must always adhere — are guaranteed to be consistent following each business operation. [Vaughn Vernon]

> In DDD, validation rules can be thought as invariants. The main responsibility of an aggregate is to enforce invariants across state changes for all the entities within that aggregate [Ref](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-model-layer-validations)
 
According to Dino Esposito
>In general terms, an invariant is a condition that always holds true in a given context. Applied to object-oriented software, an invariant indicates a condition that always evaluates to true on each instance of a class [Ref]([https://docs.microsoft.com/en-us/archive/msdn-magazine/2011/june/msdn-magazine-cutting-edge-invariants-and-inheritance-in-code-contracts])

Based on the scope there are two types of invariants. Before going to invariant let's understand what is the scope. According to Vaughn Vernon
> Each Aggregate forms a transactional consistency boundary. This means that within a single Aggregate, all composed parts must be consistent, according to business rules, when the controlling transaction is committed to the database

#### Aggregate scoped invariant
Invariant is aggregately scoped when it resides inside the aggregate boundary. We can consider the following example
Let’s we have an order domain model that contains order status and it also maintains order status history. Order status cannot be changed if it is already applied in past. This type of invariant is aggregate specific, and it resides in the aggregate boundary. We can check following example
```cs
public class Order
{
    //... Some other attributes
    public OrderStatus Status { get; private set; }
    public List<OrderStatus> StatusHistory { get; private set; }
    public void ChangeStatus(OrderStatus status)
    {
        // Invariant
        if (!StatusHistory.Any(p=>p.Status == status))
        {
            Status = status;
            StatusHistory.Add(new StatusHistory(status));
        }
    }
}
internal class StatusHistory
{
    public StatusHistory(OrderStatus status)
    {
        Status = status;
        DateCreated = DateTime.UtcNow;
    }
    public OrderStatus Status { get; set; }
    public DateTime DateCreated { get; set; }
}
```
#### Bounded context scoped invariant
Sometimes some business rule works across multiple aggregates and they are not limited to single aggregate. 
For example, user email address should be unique through the system. This invariant cannot be checked within its boundary and it requires to check all other aggregates so it crosses the aggregate boundary. Later in the post, I will try to describe how we can enforce those invariants.

### Some common scenarios
#### Should we add data validation inside the domain model or not.
Domain model should enforce business rules/invariants. Implementing data validation inside aggregate is not always necessary as we did it by deferred validation. If we want, we can use the value object, and there we can put data validation logic. But from there we cannot return the validation messages as we are throwing exceptions. 
#### Uniqueness check
Let's consider we have a user domain. Inside the system user email address should be unique. Now the question comes to mind where we will do a uniqueness check? Inside the domain model or application layer?
Actually, we can put it any one of them. 
One approach is we can check it from the application layer before invoking domain action. In this way, the domain model will be pure.

Another approach is we can introduce domain service and inject it into domain model as a method dependency then we will lose purity.
I personally like the domain model purity,  so I will do uniquness check from application layer.

Let’s consider different approach
#### Instead of uniqueness check trust on persistent level constraint
Another alternative approach is to consult with the domain expert. Do we really need this uniqueness check? How often it happens. If it happens very often then it requires a solution otherwise we can trust on database level unique constraint. If uniqueness constrain fails, then an exception will be thrown. We can add a custom exception handler that will translate this exception to a user-readable validation message.

#### Bypass uniqueness check by applying different business flow
Another alternative solution is if the business agrees we can update existing domain model by marking duplicate email found. Then that user cannot be logged in and he/she will require to reset his/her password. For details, we can check from Greg Young’s [post](http://codebetter.com/gregyoung/2010/08/12/eventual-consistency-and-set-validation). 

### Issue with always valid object
Let’s consider we have a value object it throws exception when some attribute values are null or empty. Also consider we are using event sourcing and we are storing domain event to the event store. We construct object by re-applying domain events. Now business asked us to add a new attribute to that value object and that is required or cannot be null. If we throw an exception when the value is null then what will happen for old values? We cannot construct the object from existing domain events. So it will be better if we use defer data validation flow. For details, we can check Jeffrey Palermo's [post](https://jeffreypalermo.com/2009/05/the-fallacy-of-the-always-valid-entity)

### Summary
-	Define validation logic separately and perform validation as early as possible.
-	Do business rule validation inside the domain model
-	We can enforce cross aggregate invariant from domain service or application service. I personally prefer application service because it provides domain model purity.

Finally I would like to share something from “Hands-On Domain-Driven” by Alexey Zimarev

>Logically, things such as communication protocols, user input validation, and persistence implementation are not seen as part of the domain model. These are technical and infrastructure concerns. A good rule of thumb here is that the whole domain model should be testable without involving any infrastructure. Primarily, in your domain model tests,you should not use test harnesses and mocks


