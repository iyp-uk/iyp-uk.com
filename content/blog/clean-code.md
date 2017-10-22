---
title: "Clean Code"
date: 2017-10-22T16:45:01+01:00
tags: ["best practices", "design patterns", "architecture"]
draft: false
---

This post is a summary of some notes taken from the amazing series of [clean code videos](https://www.safaribooksonline.com/library/view/clean-code/9780134661742/), themselves taken from the [clean code book](https://www.safaribooksonline.com/library/view/clean-code/9780136083238/).
Both come highly recommended if you want to improve your professionalism as a software engineer.

## Names

1. Choose names thoughtfully 
2. Communicate your intent = by reading it you understand what it is / does
3. Avoid disinformation
4. Pronounceable names
5. Avoid encodings (like Hungarian notation)
6. Ensure code can read like prose in plain english
    * Variables = nouns
    * Classes = nouns and names
    * Methods = verbs
    * booleans with isSomething
7. Name as scope dictates: 
    * Private, details functions / classes / methods / vars should have long and explicit names
    * Public, widely used functions / classes / methods / vars  should have short and explicit names

## Functions

* 1st rule: functions are small
* 2nd rule: they are smaller than that
* Functions do one thing: If a function is included in another it should be taken out
    * Extract till you drop
    * If you can extract one function from another, you should, as the function isn’t doing one thing indeed
* Classes hide in long functions => Long functions = classes

## Function structure

* Small function signatures: 3 arguments max
* No boolean argument
* Avoid switch and if statements, replace by polymorphism
* Functional programming
    * No assignment statement
    * A function is immutable: same input always gives the same output
    * CQS: Command and Query Separation
        * Command
            * Execute the command and return nothing
            * Can throw exception
        * Query
            * Execute the query and return value but do not change state
        * If a method modifies state, it must return void
        * If a method queries state, it must not modify it
    * Tell don’t ask
        * Let the object itself deal with its state rather than asking for it before asking for a command
        * Don’t chain methods
    * Block-passing technique
    * Law of Demeter 
* Structured programming
    * Sequence
    * Selection
    * Interaction
    * Single entrance at the top, single exit at the bottom, for all these blocks, and by composition modules, and systems do too

## Form

* Comments
* Classes and parameters:
    * Can have private parameters
    * Tell don’t ask implies: No getters, No setters
    * your class and the objects resulting from it are just a bunch of functions from the outside 
    * Polymorphism
    * If you expose data, make it as abstract as possible so that we can derivate from that class with no issues
* Class VS Data structure
    * DS is opposite of the class. It has public variables, and no methods
    * If there’s a switch, it’s probably because there’s a data structure hiding behind
* Use as less comments as possible, just when they’re absolutely necessary
* Average file size < 100 lines, never > 500 lines
* Use tell don’t ask
* Boundaries
    * Separate concrete things from abstract things
* Separate business objects and database with abstraction layer

## TDD

* Red > Green > Refactor, repeat
    * RED: Write failing test code, with just enough code to demonstrate a failure
    * GREEN: Write production code, with just enough code to make the test pass  (can be dumb here)
    * REFACTOR: Refactor what you’ve just written in the 2 previous phases as much as you can while keeping a green state
* TDD is like double entry bookkeeping for accountants

> Code rots because of the fear of change: only tests that you trust can eliminate that fear.

## Architecture, use cases and high level design

* Use cases / User stories
* Isolate core from periphery 

> Design delivery-mechanism independent systems

* Use cases and high level design
    * Saying architecting is choosing Java, sprint and whatnot is the same as saying a house has to be made of nails and wood
    * Architecture is not about tools, it’s about usage, 
        * see drawings of a house or a church. At first sight you understand the purpose, the intent. It has to be the same with software architecture
    * A good architecture screams use cases!
    * Don’t specify UI, databases, etc… 
        * A good architect knows how to keep options open for as long as possible
* Separation of value
    * Use cases
    * UI
    * Databases
    * Web server
    * Allows a Cost / Value analysis
* Conclusion
    * Defer decisions thanks to separation of value

### Use cases

* Describe how a user interacts with the system, but not specifying links and buttons. Otherwise it’s too related to the delivery mechanism (web in most cases). It’s called Use cases
* Primary course: Happy path
* Exception course: When something goes wrong, how should the system react?
* Books
    * Writing Effective Use Cases
    * Object Oriented Software Engineering
* What object holds the business rules?
    * See partitioning

### Partitioning

* Entities
    * Application-agnostic 
* "Interactors"
    * Application-specific, in order to achieve the goal of the use case
* Boundary objects
    * Prepare data for the delivery mechanism
    * They are the only objects which talk to the delivery mechanism

### Isolation

{{< figure src="/img/isolation.png"  >}}

### Conclusion

* Architecture
    * Does not rely on any tool or framework
    * Allows to defer decisions on tools and frameworks for a very long time
* A good architecture
    * Maximises the number of decisions not made
    * Does not depend on the delivery mechanism, even hides it
        * i.e., when you look at a system, you shouldn’t be able to tell if it’s a web system 
    * Allows the cost 
        * of the use cases
        * of the UI and other components 
        * to be evaluated independently 
    * So that their relative business value can be evaluated
* The use cases
    * Are the core of the system as they show its intent
* Interactor, Entity and Boundary partition principle
    * Interactors encapsulate use cases
    * Entities encapsulate business objects
    * Boundaries isolate the system from user interfaces
* Databases are isolated thanks to abstraction and that’s the iterator objects responsible for DB access
