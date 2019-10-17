## Object Oriented Collections API

The Data structures are procedural by nature. The API's are exposing the internal state of the data-structure. Languages like Java implements the data-structures using the Interfaces, and Abstract classes and might seem like an object oriented code for casual reader. But its not. Objects encapsulate (hides) state and expose only behavior which depends on the state. The data-structure APIs such as add, remove, get, exposes the state to the consumer. These API's are all procedural in nature and hence its very difficult to compose them to get high level behavior. If we want to synchronize the access to the list, there is no way we can achieve without introducing locks in consumer logic. One can make all the methods in List as synchronized but that wont help the consumers if they want two of the operations to be executed atomically. A typical example is contains and add done together. 

Any object oriented code returning the data-structure as its return type will also seem innocent, but it opens door for consumers to access the state of the result which should have been encapsulated in some other object.


```
interface Query {
    List<String> report(final String query);
}

Query query = Query.create(Type.SQL);
List<String> results = query.report("select * from example");
for (final String result : results) {
    System.out.println(result);
}
``` 
The above interface seems object oriented since it is only behavior which takes query string as input and returns a List of report objects. But the return type is List which will make the consumer access the state of the result directly. This makes the consumer to use imperative constructs like variable assignments, and for loops in this example. A better implementation of the above interface would be.


```
interface Query {
    Report report (final String query);
}

interface Report {
    void render (OutputStream out);
}

Query.create(Type.SQL).report("select * from example").render(System.out);
``` 

The above design doesn't expose the internal state of the report (List<String>) and wraps that inside another object (Report) which exposes behavior to write to any given output stream.

We have to do this since the List is procedural and should not be returned in object oriented API. What if we have object oriented implementation of List which doesn't expose the state of the result directly to consumers but take the logic which depends on the result in and executes them?. We can have simple interfaces such as 


```
interface Query {
    List<String> report (final String query);
}

Query.create(Type.SQL).report("select * from example").forEach(System.out::println);
``` 


Consider the following Object oriented API for List data-type.

``` 
interface List<T> {
     void forEach (final Consumer<T> action);
    
    void forEach (final int start ,final int end, final Consumer<T> action) throws IndexOutOfBoundsException;
    
    void forIndex (final int index, final Consumer<T> action) throws IndexOutOfBoundsException;
    
}
``` 

We don't have any other procedural API's like add, remove, get, in this. Since those API;s are exposing the internal state, only reasonable API for list should only visit the objects and "take" the logic which works on them instead of returning the "state" and ask the consumers to work on it.

Now we can have different implementations of this List and since the API's are concise, we can easily write multiple implementations and compose them to get high level behavior.

The different list implementations are hosted in the below github URL.

 [https://github.com/senthilganeshs/object-java.git](Link)

```
List.mapped(
     i -> fact(i), 
     List.par(
         List.of(1, 2, 3, 4, 5), 
	 Executors.newCachedThreadPool()))
    .forEach(System.out::println);
``` 

There are no variable assignments, no imperative programming constructs like if..else, for,while,do-while statements in the code. 
We wanted to compute the factorials of all values in [1,2,3,4,5] in parallel and want to print the result. The code may resemble functional style, but it is very different. The methods mapped, par, of are all static factories returning some implementation of list, and we compose the lists to get high level behavior.

The List.par takes a list of values and executes the consumer logic (System.out::println) in separate thread. Now this may seem in-efficient why would anyone want to create threads unnecessarily just to print values?. But if we see how the list is composed to List.mapped we can find that List.par is actually executing (i -> fact (i)) also in separate thread along with printing it to the console. This way List.par can be composed to any other implementation of list and can get high level behavior.

