## Coding with Constraints

We are at the cross-roads and we have no clue about the outcome of choosing one path over the other, there is 50 % chance that we chose the wrong one. 

Consider someone (a novice developer) solving a problem using object-oriented concepts finds that there are many ways to design the same problem, he too will be at the cross-roads and must choose the paths correctly to reach the destination.

> So how do we identify the right path from the wrong ones.? 

If we have constraints which helps us choose one design decision over the other, we can narrow down the choices quite easily.

In this blog post, we will identify few constraints which helps us write successful software.

What qualifies a software to be successful? 
- First it should solve the current use-case. 
- The use-case will change over time, and the software should adapt to the changing requirements easily. 
- The changes must be delivered on time so that the consumer stays with the product.

The initial version of the software which passes its tests for the given use-case works fine for the consumer. But when the requirements change, 

> How easy it is to accommodate the change? 
How do we deliver the changes within reasonable time-frame?

- Any new use case or changes to an existing use case can be accommodated easily if the software is extensible. 
- Even if the software is extensible, any new developer responsible for doing the change should be able to understand the existing code easily.(Readability)
- We can deliver the changes with confidence to the consumer only if we have (automated) tests which ensure its correctness. (Maintainability)

Still the terms are more abstract and we need to find ways to relate them in coding. We need to find what aspects of object-oriented programming can relate to these terms and what design choices we must take to maximize these attributes.?

> How to code an object which accommodates changes easily?

Objects encapsulate state and expose behavior. State is any input we pass to the constructor which creates the object. If the state of the object is substitutable then we can alter the behavior of the object easily.

In Object oriented programming we use interface to allow substitution. If the object and its state (constructor argument) is coded to its interface then the object is extensible.

If all the objects used to implement the use-case is extensible then the use-case becomes extensible.

> How to code an object which is readable?

We can draw an analogy with the natural language we use to communicate with each other. Communication is effective if we use right words in the right context. 

Substitute behavior for words and objects for context. Let’s see an example below to convey this thought.

Let’s consider simple use-case of borrowing a book from library for reading its contents. To communicate this clearly, we need words such as “borrow”, “read”, “learn” and the contexts such as “Library”, “Book”, and “Reader”.

Let’s model the context using interface and words using behavior.

```java
interface Library {
    Book book (final String isbn);
}
interface Book {
    void read (final Reader reader)
}
interface Reader {
    void learn (final String text);
}
```

Since we only code using interfaces, we can check if the code is readable by writing the consumer logic assuming the objects are available.

```java
readFromBookUseCase() {
    library.book ("978-3-16-148410-0").read (reader);
}
```
The method "book" in the context of "library" may represent obtaining a book instance by "borrowing" or "constructing" or "borrow from another library". How the instance is returned is not of importance and we only need to know that we can obtain a "book" from "library" object.

The method "read" in the context of "book"  represents furnishing the content of the book to the reader. 

The method "learn" in the context of "reader" represents consumption of contents of the book.

This code is readable since we have got right words used in the right context.

> How to code an object which is maintainable?

An object is maintainable if it has unit tests to verify each of its behavior. The tests will ensure that any changes to the behavior works as expected. The tests will also enable refactoring the code without breaking the functionality. 

We have discussed the attributes of successful software and how the below constraints help us maximize those attributes.
- Object and its dependencies should be coded against interfaces.
- Identify as many objects and behavior as required to implement the use-case which conveys the intent clearly to the reader.
- Each use-case should contain only objects and its interactions.
- Each object should accompany unit tests verifying its behavior.