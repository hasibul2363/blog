---
title: "Repository"
date: 2020-12-12
draft: false
tags : ["Repository"]
categories : ["DDD"]
---
### Background 
If weSearch through the web, lots of examples will be found for repository pattern. Different people implement it differently. There are varieties of implementations of repository pattern like a generic repository, business case specificrepository (Domain Repository), and so on. Most of the time it is found that people do redundant work like they use the generic repository and also business case specific repository but both contain similar activity/ behavior. Through the post, I will try to sharesome of my findings and thinking. Please note I am not goanna to share implementation as it is available through the web.

Through the post I will try to cover followings.
- What it is
- What problem it solves
- Domain Repository and Generic Repository when to use what
- Repository vs DAL/ DAO
- Repository and CQRS

### What it is 
Repository pattern is a conceptual framework that provides model-based communication by abstracting persistence stuff. It helps us to focus on business logic by moving persistence logic to infrastructure.

According to Martin Fowler
> A Repository mediates between the domain and data mapping layers, acting like an in-memory domain object collection. Client objects construct query specifications declaratively and submit them to Repository for satisfaction. Objects can be added to and removed from the Repository, as they can from a simple collection of objects, and the mapping code encapsulated by the Repository will carry out the appropriate operations behind the scenes. Conceptually, a Repository encapsulates the set of objects persisted in a data store and the operations performed over them, providing a more object-oriented view of the persistence layer.[[P of EAA]](https://martinfowler.com/books/eaa.html)

### What problem it solves
- Decouples application and domain layer/design from persistence.
- Provides model-based interface.
- Easy substitution of dummy implementation, for use in testing, using in-memory collection

### Domain Repository and Generic Repository when to use what
#### Domain Repository
It is directly associated with the business. We will add behavior based on ubiquitous language. It is not something like Add, Update, Delete, or Get operation. Consider an example of IOrderRepository. It is a place where we will add business behavior or query like GetCustomMadeOrders. To collect custom-made orders; it requires to perform some filtering on order collection and filtering logic is complex. So in that case we will use domain repository like following
```cs
IOrderRepository
{
    IEnumerable<Ordder> GetCustomMadeOrders();
}
```
> Please note this interface should be defined inside Domain layer and its implementation will go to infrastructure layer. We should not implement inside domain layer.
#### Generic Repository
This repository is not business centric. Here operations are common like Add, update, delete and read. And this repository works on any type of object. If we have some model or business where we don’t have any complex logic on that cases we can easily use it. Like we want to store city name in our system and it requires only crud operation. On that time we don’t need to crate separate repository for it. We can directly use Generic Repository to perform crud operations.

I have seen many people do same thing twice like they have generic repository and also model specific repository but both did the same. Like following
```cs
IRepository<T>
{
    void Add(T model);
    T Read(guid id);
}
```
```cs
// redundant
IUserRepository
{
    void Add(User model);
    User Read(guid id);
}
```
```IUserRepository``` is redundant because we are doing same thing and not writing any custom business logic to it. We can easily drop IUserRepository and it can be replaced by generic repository.

We can apply method level or class level generics. When we are talking about Generic Repository, actually it is a tool or library that helps to abstract the persistent layer. As it is not a business repository easily we can apply generics to method level instead of class level. Something like following. It will reduce unnecessary instantiation of type based repository.

```cs
IRepository
{
void Add<T>(T Add);
T Read<T>(guid id);
}
```
### Repository vs DAO or DAL
Repository works with the model where all invariants are mate. on the other hand, DAL works with the table or dataset. DAL is an old concept where we ware developing system based on table and stored procedure. If we have database sentric system then DAL is ok and it is not object orientd way of communication. DAL also violates encapsulation. If we use DAL then there is a possibility we can violate encapsulation as we can access interla object without the aggregate root. If we do so then there is a chance it will not meet all invariants.
### Repository and CQRS
Repository works with the collection and that collection is the same type. It works on single aggregate root. If there is a case, we need to communicate with multiple aggregates and also need to return some custom type model in that case query or query handler approach is good. 
### Summary
Need to create separate repository class when we are doing DDD and business has specific use case.

Use Generic repository when we are doing only CRUD operation. It is not necessary to create separate repository class for each type. It is redundant. Create separate repository class only when you have to implement some custom logic.

Go for CQRS or query model-based approach when it requires to query through multiple domain models to fulfill the business requirement. Repository only works on same type of domain model.
