---
title: "Design Patterns"
date: 2017-08-12T16:43:22+01:00
draft: true
tags: [ "design patterns" ]
---

## Coverage

You'll find here a descriptive list of the most common design patterns in programming.
[Wikipedia](https://en.wikipedia.org/wiki/Design_Patterns) provides a list of more design patterns for you to read.

## Prerequisites

We assume you're already familiar with [Object Oriented Programming](https://en.wikipedia.org/wiki/Object-oriented_programming).
Even if you are, this link can be read quickly and could refresh your thoughts on OOP effectively.

## Types of design patterns

### Creational 

> Creational patterns are ones that create objects for you, rather than having you instantiate objects directly. 
> This gives your program more flexibility in deciding which objects need to be created for a given case.

[Read more on Wikipedia](https://en.wikipedia.org/wiki/Creational_pattern)

### Structural 

> These concern class and object composition. 
> They use inheritance to compose interfaces and define ways to compose objects to obtain new functionality.

[Read more on Wikipedia](https://en.wikipedia.org/wiki/Design_Patterns#Structural)

### Behavioural 

> Most of these design patterns are specifically concerned with communication between objects.

[Read more on Wikipedia](https://en.wikipedia.org/wiki/Design_Patterns#Behavioral)

## Design patterns

| Design pattern | Type | Typical use cases |
|---|---|---|
| [Strategy pattern](#strategy-pattern) | Behavioral | Allows one of a family of algorithms to be selected on-the-fly at runtime |
| [Pub / Sub = Observer pattern](#observer-pattern) | Behavioural | Loose coupling |
| [Decorator pattern](#decorator-pattern) | Structural | Add / Override responsibilities to objects without modifying the classes |
| [Factory pattern](#factory-pattern) | Creational | Reduce dependencies |
| [Singleton pattern](#singleton-pattern) | Creational | Restricts object creation to a single one per class |
| Command pattern | Behavioural | Encapsulate invocation: Logging or Queueing |
| Adapter and Facade patterns | Structural | Adapt a design expecting an interface to a class that implements a different one |
| Template Method pattern | Behavioural | Encapsulate *pieces of algorithms* so that subclasses can hook onto it |
| Iterator and Composite patterns | Behavioural / Structural | Manage collections |
| State pattern | Behavioural | Allows an object to alter its behavior when its internal state changes |
| Proxy pattern | Structural | Control and manage access |
| Compound pattern | N/A |Patterns of patterns: compose patterns together |

### Strategy pattern

> The Strategy Pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. 
Strategy lets the algorithm vary independently from clients that use it.

{{< figure src="/img/strategy-pattern.png" title="Strategy pattern" >}}

More tools in our design toolbox!

{{< figure src="/img/strategy.png" title="What we've learned so far" >}}

### Observer pattern

It's the pub/sub mechanism, a very popular pattern. 
The principle is the same as a newspaper and its subscribers.
They can subscribe which will result in being delivered with news regularly, and unsubscribe to stop receiving them.

* The publisher is called the **Subject** or **Observable**
* The subscriber is called **Observer**

> The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, 
all of its dependents are notified and updated automatically.

{{< figure src="/img/observer.png" title="What we've learned so far" >}}

### Decorator pattern

>  The Decorator Pattern attaches additional responsibilities to an object dynamically. 
Decorators provide a flexible alternative to subclassing for extending functionality.

{{< figure src="/img/decorator.png" title="What we've learned so far" >}}

### Factory pattern

Factories handle the details of object creation.
 
> The Factory Method Pattern defines an interface for creating an object, but lets subclasses decide which class to instantiate. 
Factory Method lets a class defer instantiation to subclasses.

{{< figure src="/img/factory-method.png" >}}

#### Abstract factory pattern

> The Abstract Factory Pattern provides an interface for creating families of related or dependent objects without specifying their concrete classes.

Here's the diagram you should end up with when using the abstract factory pattern:

{{< figure src="/img/pizza-diagram-abstract-factory.png" title="Abstract factory diagram" >}}

{{< figure src="/img/factory.png" title="What we've learned so far" >}}

### Singleton pattern

This is really one of a kind! With this pattern, you can only create one and **only one** instance.
If you wonder why it's useful, think about a logger or a registry, you don't want to have multiple copies.

## Credits

This article is made of notes from several design pattern sources, but good chucks are taken from [Head first design patters](https://www.amazon.co.uk/Head-First-Design-Patterns-Freeman/dp/0596007124) which presents these patterns in an interesting manner and is worth reading.