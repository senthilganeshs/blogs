## Coding with Constraints

We are at the cross-roads and we have no clue about the outcome of choosing one path over the other, there is 50 % chance that we chose the wrong one. 

Consider someone (a novice developer) solving a problem using object-oriented concepts finds that there are many ways to design the same problem, he too will be at the cross-roads and must choose the paths correctly to reach the destination.

> So how do we identify the right path from the wrong ones.? 

If we have constraints which helps us choose one design over the other, we can narrow down the choices quite easily.

In this blog post, we will identify few constraints which will helps us write successful software.

What qualifies a software to be successful? 
- First it should solve the current use-case. 
- The use-case will change over time, and the software should adapt to the changing requirements easily. 
- The changes must be delivered on time so that the consumer stays with the product.

The initial version of the software which passes its tests for the given use-case works fine for the consumer. But when the requirements changes, 

> How easy it is to accommodate the change and deliver on time?

- Any new use case or changes to an existing use case can be accommodated easily if the software is extensible. 
- Even if the software is extensible, any new developer responsible for doing the change should be able to understand the existing code easily.(Readability)
- We can deliver the changes with confidence to the consumer only if we have (automated) tests which ensure its correctness. (Reliabiliy / Maintainability)

The software is exensible if the usecase(s) which the software provides are all extensible. The use-case is extensible if all the objects used to implement the use-case is extensible. 

Hence we need to understand how an object can be extensible and maintainable in-order to cascade these attributes to the use-cases and then to the software itself.

The rest of the blog discusses how the decisions taken at the object level defines extensibility, readability and maintainability (or reliability) of the software.

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

> How to code an object which accommodates changes easily?

In Object oriented programming we use interface to allow substitution. If the object is coded to its interface then it can be replaced with another implementation. The code which depends on an interface for behavior is extensible since we can substitute any implementation to it.

If the use-case is implemented only using objects and its interaction with other objects, then the use-case itself becomes extensible.

Let's write pseudo implementation of the above interfaces to analyze the extensibility of our design.

```java
readFromBookUseCase() {
    library.book ("978-3-16-148410-0").read (reader);
}
library (isbnToBooksMap) {
    Book book (final String isbn) {
        return isbnToBooksMap.get(isbn);
    }
}
book (contents) {
    void read (Reader reader) {
        contents.forEach (text -> reader.learn (text));
    }
}
reader() {
    void learn (final String text) {
        System.out.println ("Learning "+ text);
    }
}
```
The behavior "book" depends on string attribute ("isbn") which makes the behavior difficult to extend. But Library object is extensible and hence can have different implementations. Library object which finds book by author, title, or by any other parameter can be separate implementations of Library interface. 

We are still tied to the fact that the locator is a "String" attribute which is primitive. But if we know the parameters to lookup a book can always be represented as a "String" then we are good. 

The behavior "learn" in the Reader object again depends on primitive "String" for the content. If we get new use-case to support learning from images, tables, or charts then we will be in trouble. Let's fix that by changing the "String" argument to Content object.

```java
interface Reader {
    void learn(final Content content);
}
```
Now what behavior we should add to the Content type we have invented? The existing implementation of the "Reader" object gives us the clue. The old implementation prints the text to the console. Let's add that as a behavior to the Content object.

```java 
interface Content {
    void print (final OutputStream os);
}
textContent (text) {
    void print (final OutputStream os) {
        os.write (text);
    }
}
tableContent (header, rows) {
    void print (final OutputStream os) {
        // print header and rows.
    }
}
reader (final OutputStream os) {
    void learn (final Content content) {
        content.print (os);
    }
}
```
We can now see that the reader object is decoupled from knowing the details of book contents. We can now easily support the use-case of having "text", "image", "table" or "charts" content within the same book and can be consumed by the same reader object. The knowledge of interpretting the content is confined within the content object.

Let's see final re-write of all the interfaces and pseudo-implementations.

```java
interface Library {
    Book book (final String locator);
}
interface Book {
    void read (final Reader reader);
}
interface Reader {
    void learn (final Content content);
}
interface Content {
    void print (final OutputStream os);
}

isbnLibrary (isbnToBookMap) {
    Book book (String isbn) {
        return isbnToBookMap.get(isbn);
    }
}
storyBook (contents) {
    void read (final Reader reader) {
        contents.forEach (content -> reader.learn(content));
    }
}
textContent (text) {
    void print (final OutputStream os) {
        os.write (text);
    }
}
tableContent (header, rows) {
    void print (final OutputStream os) {
        // print header and rows.
    }
}
bookReader (OutputStream os) {
    void learn (final Content content) {
        content.print (os);
    }
}
```

We have seen how our original design was difficult to extend when
- We want to look-up book by different parameters apart from isbn.
- We want to learn different contents such as text, tables, charts, etc..,

We chose providing separate implementations of library object for each lookup parameter (which is isbn, author, title, and so on) and new object to replace non-object parameter (text) in the "learn" method

We also noticed that the code becomes more extensible when we replace new objects in-place of non-objects (primitives and data-structures).

> How to code an object which is maintainable?

An object is maintainable if it has unit tests to verify each of its behavior. The tests will ensure that any changes to the behavior works as expected. The tests will also enable refactoring the code without breaking the functionality. 

To summarize we have identified below constraints while implementing the use-case.
- Object and its dependencies should be coded against interfaces.
- Identify as many objects and behavior as required to implement the use-case which conveys the intent clearly to the reader.
- Behavior of an object is extensible if it depends on objects.
- Each use-case should contain only objects and its interactions.
- Each object should accompany unit tests verifying its behavior.

Should every object oriented system follow above constraints always.? Need not be.. It largely depends on the domain knowledge and ability to forecast the possibile changes the software will go through. Do we need to follow this exercise everywhere in the code? That would be ideal but wherever we expect the changes to happen, it becomes essential.

For Example, an enterprise application which followed strict object-oriented programming principles for its infrastructure will make the infrastructure substitutable. This means that the same application which runs in one environment can be ported to another environment easily without impacting its domain logic.