## Functional data-structures in JAVA

This is in continuation with my previous post [Functional Programming in JAVA](https://senthilganesh.hashnode.dev/functional-programming-in-java-ck27fcnkq001od0s1zfiincl2) where we concluded saying choosing functional style of programming in JAVA will break object oriented programming principles. 

But with functional programming we get lots of APIs for free by implementing very few functions. For Example, a data-structure implementing foldr function can get isNull, length, elem, maximum, minimum, sum, product, etc.., for free. This is very useful for any programming paradigm.

Losing those capabilities in object oriented paradigm will be a loss. We need to understand how to transform these functional concepts into object oriented coding without violating its principles. 

Data-structures in functional languages allows one to pattern match the shape of the datatype (internal state). The functional data-structures are persistent data-structures which needs to construct and de-construct the data-types for any operation involving modifications. If we know the shape of the data-type and constructor to invoke, we can construct data-types on the fly.

**This is a violation of OOPS principle (encapsulation) and hence we can't do that in Object oriented languages**.

If we don't want to expose the shape of the data-type then we need to expose a behavior which constructs new data-structure accepting given input value.

```java
interface Iterable<T> {
    <R> R empty();
    Iterable<T> build(final T input);
    <R> R foldLeft (final R seed, final BiFunction<R,T,R> fn);
}
```

- 'empty' represents an empty instance of the data-structure.
- 'build'  returns a new data-structure after accepting the new input value.
- 'foldLeft' combines seed with values present in the data-structure in the left to right order.

For re-constructing the data-structure, we would want to start with empty data-structure and keep putting values in to it incrementally. The methods 'empty' and 'build' helps with that. 

Each data-structure knows how to visit values within. For Example, different tree traversals are possible for a single binary tree and each of them visits the values differently. Hence having the foldLeft (iterate) method from the implementation will abstract us from knowing the details of how the values are managed by any data-structure.

foldLeft implementation is again very simple. For example, lets see the implementation of foldLeft for arrays.

```java
@Override <R> R foldLeft (final R seed, final BiFunction<R,T,R> fn) {
    R start = seed;
    for (final T t : values) {
        start = fn.apply(start, t);
    }
    return start;
}
```

Functional languages don't have this level of generalization of data-types, and they rely on pattern matching the shape (internal state) of the data-structure and should know the constructor to call to create new data-structures.

Let's see how we can implement more API's consuming only these three functions. All these methods are default methods in the Iterable<T> interface.

```java
default <R> R foldRight (final R seed, final BiFunction<T,R,R> fn) {
    final Function<R, R> res = 
        foldLeft(a -> a,
            (g, t) -> s -> g.apply(fn.apply(t, s)));
    return res.apply(seed);
}
```
foldRight combines the seed with right most value in the data-structure and reduces the data-structure in right to left order. We can implement foldRight using foldLeft, by wrapping the seed value and the intermediate results inside a function instead of using it as is. When we apply the seed to the final function we got from the foldLeft, it gets applied to the right most value.

```java
default Iterable<T> filter (final Predicate<T> pred) {
    return foldLeft (
        empty(),
        (r, t) -> pred.test(t) ? r.build(t) : r);
}
```
We iterate the data-structure and specify the start value as empty(). We build the data-structure only if the value satisfies the predicate. If not, we simply skip the value and return the data-structure as-is.

```java
default <R> Iterable<R> map (final Function<T, R> fn) {
    return foldLeft (
        empty(),
        (rs, t) -> rs.build(fn.apply(t)));
}
```
Map is similar to filter implementation except that it applies function fn to each value in the data-structure and construct a new one.

```java
default Iterable<T> concat (final Iterable<T> first) {
    return foldLeft (first, (ts, t) -> ts.build(t));
}
```
concat passes first as seed so all values in the current data-structure is added  to first and we get new data-structure which is concatenated with values of first and current data-structure.

```java
default <R> Iterable<R> flatMap (final Function<T, Iterable<R>> fn) {
    return foldLeft (
        empty(),
        (rs, t) -> fn.apply(t).concat(rs));
}
```
flatMap takes a function returning new data-structure for given value in the first data-structure. We need to concatenate the results after applying the function to flatten it.

Functional languages like Haskell imposes a restriction that Iterable<R> specified in the flatMap API parameter and return type to be same data-structure. But in our implementation, it will work for any implementation of iterable since we don't deal with shape of the data-structure we are not restricted by this constraint.

```java
default <R> Iterable<R> apply (final Iterable<Function<T, R>> fns) {
    return fns.flatMap(f -> map(f));
}
```
We have functions encapsulated inside one data-structure and we wanted to apply them to all values in other data-structure, we can use apply method. 
For example,

```bash
> Maybe.some(5).apply(List.of(i -> i - 1, i -> i + 1));
> [4, 6]
```

```java
default <R, S> Iterable<S> liftA2 (
        final Function<T, Function<R, S>> fn, 
        final Iterable<R> rs) {
    
    return rs.apply(map(fn::apply));        
}
```
map and apply functions we saw above are so commonly used together and hence we have liftA2 function, which takes a function taking values from two different data-structures and returns a new data-structure of type rs containing the results. For Example

```bash
> Array.of(1,2,3).liftA2((i, r) -> i + r, Maybe.some(10))
> [11, 12, 13]

> Array.of(1,2,3).liftA2((i, r) -> i + r, Array.of(10, 20))
> [11, 21, 12, 22, 13, 23]
```
We are taking values from two different datastructures, combine them using function ((i, r) -> i + r) and construct the results using first data-structure (Array).

Our next function is traverse. This is similar to flatMap except that it preserves the structure instead of flattening it out. 

```java
default <R> Iterable<Iterable<R>> traverse (
                     final Function<T, Iterable<R>> fn) {            
    Iterable<Iterable<R>> seed = empty();
    Iterable<Iterable<R>> sseed = seed.build(empty());
       
    return foldLeft (sseed, 
                (rrs, t) -> 
                fn.apply(t).liftA2((r,  rs) -> rs.build(r), rrs));
}
```
For Example,
```bash
> Array.of(1,2,3).traverse(i -> Maybe.some(i + 1))
> Some ([2, 3, 4])
```

For the sake of understanding lets substitute the data-structures inplace of iterable in the traverse function.

- seed, sseed => Maybe<Array<Integer>>
- fn                => Function<T, Maybe<Integer>>
- (t, rrs)          => Integer, Maybe<Array<Integer>>
- fn.apply(t)   => Maybe<Integer>
- (r, rs)           =>  Integer, Array<Integer>

For every value 't' in the Array  (in right to left order), we apply the function 'fn' and get Maybe<Integer>.  Now we wanted to construct a list inside the Maybe data-structure and we already got Maybe containing the result. Now we can call liftA2 to construct the value inside another data-structure (Array).

```java
static <R> Iterable<Iterable<R>> sequence (
        final Iterable<Iterable<R>> rs) {
    return rs.traverse(ir -> ir);
}
```

If we wanted to only de-construct and re-construct the values from one data-structure to another, we can use sequence instead of traverse. Lets see an example.

```bash
> Iterable.sequence(Array.of(Maybe.some(1), Maybe.some(2), Maybe.some(3)));

> Some ([1, 2, 3])
```

Here we are de-constructing the Maybe datastructure and re-construct it with all the values combined in a list. This is useful because if there is a Maybe value in the list which is nothing then the result will be nothing.

```bash
> Iterable.sequence(Array.of(Maybe.some(1), Maybe.some(2), Maybe.some(3), Maybe.nothing()))

> Nothing
```

A point to be noted here is that since functional languages depends on pattern matching the shape of the data-structure and should know the constructor to call, they cannot generalize the implementation like we have done here. 

For Example, one needs to implement traverse to get sequence for free or vice-versa. But in our object oriented implementation we get all the above functions for free.

We can now define whole bunch of API's for any data-structures only with the help of above functional API's and all the implementations of the Iterable<T> interface will get them for free.

Now lets see what it takes to implement Iterable

```java
interface List<T> extends Iterable<T> {
    /* default List functions implemented using the functional APIs of Iterable*/
}

final static class ListIterable<T> implements List<T> {

    private final java.util.List<T> list;

    ListIterable(final java.util.List<T> list) {
        this.list = list;
    }
        
    @Override
    public <R> Iterable<R> empty() {
        return new ListIterable<>(new ArrayList<>());
    }

    @Override
    public Iterable<T> build(T input) {
        java.util.List<T> newList = new ArrayList<>(list);
        newList.add(input);
        return new ListIterable<>(newList);
    }

    @Override
    public <R> R foldLeft(R seed, BiFunction<R, T, R> fn) {
        return list.stream().reduce(seed, (r, t) -> fn.apply(r, t), (r1, r2) -> r2);
    }  
}
```
This implementation may not be performant, but it works. If we look at the implementation there is no violation of encapsulation and it can get all the above APIs for free. All the APIs are implemented as default methods in the Iterable<T> interface. 

Interface defining default methods can result in "Fragile base class" if derived class overrides them consuming other API's which inturn depend on this method in the default implementation. Since we don't have hierarchy of classes, the complexity is greatly reduced.

Lets consider the same CoffeeShop example, where we did below for a single order of Coffee

```java
inventory.order("Mocha", 1).bill(credit)
.ifSuccess (PaidItem::checkOut)
.ifFailure(up -> up.revert(inventory));
```

Now lets say we wanted to order multiple servings of different Coffee, then we can write as follows :

```java
List.of("Mocha", "Espresso", "Latte") //list of coffees
    .liftA2(
        //combine two lists into list of orders
        (item, qty) -> inventory.order(item, qty), 
        List.of(1,2,1)) //number of servings
    .forEach(
        order -> 
            order.bill(credit)
            .ifSuccess(PaidItem::checkOut)
            .ifFailure(up -> up.revert(inv)));
```
We still have objects modelling our use-case but we have eliminated the imperative code involving reading from two lists and construct collection of order objects.

This code is still object oriented even though it uses functional data-structures and functional API's. They are all used in the context of avoiding imperative coding involving loops, recursion, variable assignments, etc.., rather than modelling the problem itself.

You can find the github project implementing functional apis  [here](https://github.com/senthilganeshs/functional-java) 

Please leave your comments below. Happy Coding. !!!