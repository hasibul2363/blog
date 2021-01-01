---
title: "Repository"
date: 2020-12-12
draft: true
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
