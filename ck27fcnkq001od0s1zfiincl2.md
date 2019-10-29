## Functional Programming in Java

With functional wave hitting every programming language across different paradigms, every language is adopting some new constructs to support functional style programming. But is it enough?. Do we need to write functional code to ensure correctness and clarity? We are going to find out by writing functional code in java for a simple CofeeShop Application.

CoffeeShop has a menu of different varieties of coffee. There is fixed limit of servings and price for each item. Customers can order coffee and if payment is successful, the inventory and credits for customer is updated. For unsuccessful payment the items are shelved back to the store.

In Functional programming, there is no notion of Object. So we are going to follow below conventions in this blog post

- Class will be used to represent state and all states are immutable. 
- Interfaces are used as namespace for functions for clarity. 
- When a datatype has different constructors (this constructor is very different from the one we see in object oriented code) we use interface for substitution. 
- All functions should be referentially transparent. Referentially transparent functions are ones which doesn't have side-effect and returns the same result for given inputs when involved any number of times.

With the above guidelines let’s start writing functional code in JAVA

All the states are represented using class with immutable fields. No behaviour should be associated with state. 

```java
class Item {
    final String name;
    final double price;
    final int qty;
    Item (final String item, final double price, final int qty) {
        this.name = item;
        this.price = price;
        this.qty = qty;
    }
}
class PaidItem {
    final Item item;
    PaidItem (final Item item) {
        this.item = item;
    }
}
class UnPaidItem {
    final Item item;
    UnPaidItem (final Item item) {
        this.item = item;
    }
}
class Credit {
    final double balance;
    Credit (final double balance) {
        this.balance = balance;
    }
}
```

Item represents a menu in the Coffee Shop. 
PaidItem and UnPaidItem represents the items for which payment was successful and unsuccessful res.., 
Credit represents the balance amount which can be used for payment.

Before we implement our use-case, lets get our basics right.

We are going to see fold function which is used to implement almost all API's in this blog post. fold is similar to iterate method we have seen in this  [post](https://senthilganesh.hashnode.dev/array-operations-and-recurrence-in-object-oriented-programming-ck1z3xiop00w19ss1b4kj431n) 

fold is used for iteration of collection of items. Lets see how it can be implemented for List data type.

```java
R fold (final BiFunction<R,T,R> fn, final R seed, final List<T> list);
```

First lets see an imperative version of the fold method.

```java
R fold (final BiFunction<R,T,R> fn, final R seed, final List<T> list) {
    R start = seed;
    for (final T i : list) {
        start = fn.apply (start, i);
    }
    return start;
}

//now the recursive version of it is 
R fold (final BiFunction<R,T,R> fn, final R seed, final List<T> list) {
    return fold (fn, fn.apply(seed, list.head), list.tail);
}
```
This is normal iteration we do in our day to day programming. The logic to include the ith value to the result is expressed as a function, so it can be used for variety of different implementations. 

For example if we wanted to filter the list based on some predicate then, we can write the filter function using the fold as follows:

```java
List<T> filter (final Predicate<T> pred, final List<T> list) {
    return fold ( (ts, t) -> pred.test(t) ? List.cons(t, ts) : ts, List.empty(), list);
}

//Lets understand the imperative version of above filter implementation.
List<T> filter (final Predicate <T> pred, final List<T> list) {
    List<T> start = List.empty(); //start with empty list
    for (T i : list) {
        if (pred.test(i)) { //if ith value satisfies the predicate.
            start = List.cons (i, start); //include ith value to the list.
        } /*else dont include the ith value to the list.*/
    }
    return start;
}

```
We can see from the above examples that the fold function provides only the mechanism to iterate the list. The logic which should go within the iteration is specified by the "BiFunction<R,T,R> fn" and "R seed" parameters. Using just these two parameters we can inject any logic within the iteration. 

Thats enough functional programming 101 to get through the rest of the blog.

Lets split our usecase of ordering coffee into four highlevel API's
- **order **- specify the item and quantity to place an order. It involves reducing the servings available in the store by the specified quantity.
- **bill **- generate bill for the above order.
- **pay  **- pay for the ordered item(s) by reducing the amount from the credit.
- **shelve **- addup the servings deducted for the unpaid items. 

```java
interface CoffeeShop {

    static Tuple<List<Item>, List<Item>> order (
              final List<Item> store,
              final String name,
              final int qty) {
        
        return ListOps.foldl (
             (tis, item) -> {
                 if (item.qty > qty && item.name.equalsIgnoreCase (name))
                     return new Tuple<>(
                         List.cons (new Item (item.name, item.price, qty), tis._1),
                         List.cons (new Item (item.name, item.price, item.qty - qty), tis._2));
                  else 
                       return new Tuple<>(tis._1, List.cons (item, tis._2));
             },
             new Tuple<>(List.empty(), List.empty()),
             store);
    }
}
```

We order an item by specifying the name and qty. If the item is present in the store and the qty is less than the available servings, then order can be made. The order returns tuple of List of items ordered and list of items in the inventory (after deducting the ordered quantity). This is needed since functional program is referentially transparent which means if we execute the same method with same parameters again and again we should get the same result always.

We have made an order, now we need to bill the items. Lets write our bill method ..

```java
interface Billing {
    static Function<Credit, Tuple<Credit, Either<PaidItem, UnPaidItem>>> bill (final Item item) {

        return credit -> {
            if (Payment.pay (credit, item.price * item.qty) == Maybe.<Credit>nothing()) 
                return new Tuple<> (credit, Either.right (new UnPaidItem (item)));
            else
                return new Tuple<> (new Credit (credit.balance - item.price * item.qty),
                    Either.left (new PaidItem (item)));
        }
    }
}

```

Our bill method is little tricky, It returns a function which accepts a Credit and then returns a tuple of Credit (with balance deducted upon successful payment) and Either of paid or unpaid item. We have got the bill now and we need to perform the payment.

```java
interface Payment {
    static Maybe<Credit> pay (final Credit credit, final double price) {
        if (credit.balance - price > 0) 
            return Maybe.some (new Credit (credit.balance - price));
        return Maybe.nothing();
    }
}
```
If you remember our billing function, we deducted the qty from the store to block the item for the customer. If the payment is not made then we have to shelve the items back to the store. 

```java
interface CoffeeShop {
    static List<Item> shelve (final List<Item> store, final List<UnPaidItem> items) {
        return ListOps.foldl (
            (is, i) -> 
            List.cons (
                 ListOps.foldl(
                      (ii, ui) -> ui.item.name.equalsIgnoreCase (i.name) ? 
                                     new Item(i.name, i.price, i.qty + ui.item.qty) : ii ,
                       i,
                       items),
                  is),
              List.empty(),
              store);                
    }
}
```

You might have observed that the data structures we have used here are different from the ones we use from JDK library. For functional programming we cannot use datastructures which depends on state mutation. So we need to define new functional data structures for writing functional code. 

So far we have seen List, Tuple, Either, and Maybe data types. Let’s see the implementations of these data structures before we can stitch the functions to implement our use-case.

# Functional Data-structures

## List
List is a simple recursive data-structure which has head(which is a value) and a tail (which is another list). Empty represents an empty list. To make the types Cons and Empty substitutable, we are making use of inheritance.

```java

interface List<T> {
    
    static <R> R head (final List<R> list) {
        if (list == List.EMPTY)
            throw new UnsupportedOperationException("empty list");
        return ((List.Cons<R>) list).head;
    }
    
    static <R> List<R> tail (final List<R> list) {
        if (list == List.EMPTY)
            return list;
        return ((List.Cons<R>) list).tail;
    }
    
    static <R> List<R> of (final R ...values) {
        if (values == null || values.length == 0)
            return empty();
        List<R> list = empty();
        for (R value : values) {
            list = cons (value, list);
        }
        return list;
    }

    final static class Cons<T> implements List<T> {

        private final List<T> tail;
        private final T head;

        Cons (final T head, final List<T> tail) {
            this.head = head;
            this.tail = tail;
        }

        @Override public String toString() {
            return head + "," + tail.toString();
        }
    }
    static final List<Void> EMPTY = new Empty<>();

    static<R> List<R> empty() {
        return (List<R>) EMPTY;
    }

    static <R> List<R> cons (final R head, final List<R> tail) {
        return new Cons<>(head, tail);
    }

    final static class Empty<T> implements List<T> {
       @Override
        public String toString() {
            return "[]";
        }
    }
}
```

## List Operations

```java

interface ListOps {
    
    static <T, R> R foldl (final BiFunction<R,T,R> accum, final R seed, final List<T> lst) {
        if (lst == List.EMPTY)
            return seed;
        return foldl (accum, accum.apply(seed, List.head(lst)), List.tail(lst));
    }

    static <T, R> List<R> map (final Function<T, R> fn, final List<T> lst) {
        return foldl ((ts, t) -> List.cons(fn.apply(t), ts), List.empty(), lst);
    }
    
    static <T> List<T> concat (final List<T> left, final  List<T> right) {
        return foldl ((ts, t) -> List.cons(t, ts), left, right);
    }
    
    static <T, R> List<R> flatMap (final Function<T, List<R>> fn, final List<T> lst) {
        List<List<R>> listOfLists = foldl ((rss, t) -> List.cons(fn.apply(t), rss), List.empty(), lst);
        return foldl ((rs, rs1) -> concat(rs, rs1), List.empty(), listOfLists);
    }

    static <T> List<T> filter (final Predicate<T> p, final List<T> lst) {
        return foldl ((ts, t) -> p.test(t) ? List.cons(t, ts) : ts, List.empty(), lst);
    }
    
    static <T, R> List<R> apply (final List<Function<T, R>> fns, final List<T> lt) {
        return foldl (
            (rs, t) -> foldl((rrs, fn) -> List.cons(fn.apply(t), rs), List.empty(), fns),
            List.empty(),
            lt);
    }
}

```
We are using fold here to implement map, concat, flatMap, filter and apply functions.

## Either

Object implement of this data type is used in this  [post](https://senthilganesh.hashnode.dev/exception-handling-object-oriented-way-cjzlbrzw3000wwks1n2909li1). It represents presence of either of values A and B at any point of time.

```java
interface Either<A, B> {

    final static class Left<A, B> implements Either<A, B> {
        public final A left;

        Left (final A left) {
            this.left = left;
        }
        
        @Override
        public String toString() {
            return left.toString();
        }
    }

    final static class Right<A, B> implements Either<A, B> {
        public final B right;

        Right (final B right) {
            this.right = right;
        }
        
        @Override
        public String toString() {
            return right.toString();
        }
    }       

    public static <R, S> Either<R, S> left (final R value) {
        return new Left<>(value);
    }

    public static <R, S> Either<R, S> right (final S value) {
        return new Right<>(value);
    }
}

```

## Either Operations

```java
interface EitherOps {

    static <T, R, S> S foldlL (final BiFunction<S,T,S> accum, final S seed, final Either<T, R> either) {
        if (either instanceof Either.Left) {
            return accum.apply(seed, ((Either.Left<T, R>) either).left);
        }
        return seed;
    }
    
    static <T, R, S> S foldRL (final BiFunction<S,R,S> accum, final S seed, final Either<T, R> either) {
        if (either instanceof Either.Right) {
            return accum.apply(seed, ((Either.Right<T, R>) either).right);
        }
        return seed;
    }
    
    static <T, R, S> Either<T, S> mapR (final Function<R, S> fn, final Either<T, R> either) {
        return (either instanceof Either.Left) ? Either.left(((Either.Left<T, R>) either).left) : Either.right(fn.apply(((Either.Right<T, R>) either).right));
    }

    static <T, R, S> Either<S, R> mapL (final Function<T, S> fn, final Either<T, R> either) {
        return (either instanceof Either.Left) ? Either.left(fn.apply(((Either.Left<T, R>) either).left)) : Either.right(((Either.Right<T, R>) either).right);
    }        
}
``` 

## Maybe

```java
interface Maybe<T> {
    
    static <R> R unsafe (final Maybe<R> maybe) {
        if (maybe == Maybe.NOTHING) {
            throw new UnsupportedOperationException("nothing");
        }
        return ((Some<R>) maybe).value;
    }

    final static class Some<T> implements Maybe<T> {
        private final T value;

        Some (final T value) {
            this.value = value;
        }
        
        @Override
        public String toString() {
            return value.toString();
        }
    }

    static final Maybe<Void> NOTHING = new Nothing<>();

    static<R> Maybe<R> some (final R value) {
        return new Some<>(value);
    }

    static <R> Maybe<R> nothing () {
        return (Maybe<R>) NOTHING;
    }

    final static class Nothing <T> implements Maybe<T> {
        @Override
        public String toString() {
            return "";
        }
    }
}

## Maybe Operations
```java
interface MaybeOps {
    static <T, R> R foldl (final BiFunction<R,T,R> accum, final R seed, final Maybe<T> maybe) {
        if (maybe == Maybe.NOTHING)
            return seed;
        return accum.apply(seed, Maybe.unsafe(maybe));
    }

    static <T, R> Maybe<R> map (final Function<T, R> fn, final Maybe<T> maybe) {
        return (maybe == Maybe.NOTHING) ? Maybe.nothing() : Maybe.some(fn.apply(Maybe.unsafe(maybe)));
    }
    
    static <T, R> Maybe<R> flatMap (final Maybe<T> ma, final Function<T, Maybe<R>> fmb) {
        if (ma == Maybe.NOTHING) 
            return Maybe.nothing();
        return fmb.apply(Maybe.unsafe(ma));
    }

    static <T> Maybe<T> filter (final Predicate<T> p, final Maybe<T> maybe) {
        if (maybe == Maybe.NOTHING)
            return maybe;
        return p.test(Maybe.unsafe(maybe)) ? maybe : Maybe.nothing();                
    }
}
```
## Tuple

```java
class Tuple<A, B> {
    public final B _2;
    public final A _1;

    Tuple (final A a, final B b) {
        this._1 = a;
        this._2 = b;
    }
    
    @Override
    public String toString() {
        return "(" + _1 + "," + _2 + ")";
    }
}
```
##Tuple operations

```java
interface TupleOps {
    
    static <T, R> Tuple<List<T>, List<R>> fromList (final List<Tuple<T, R>> list) {
        return 
            ListOps.foldl(
                (ttsrs, ttr) -> 
                    new Tuple<>(
                                     List.cons(ttr._1, ttsrs._1), 
                                     List.cons(ttr._2, ttsrs._2)),
                new Tuple<>(List.empty(), List.empty()),
                list);            
    }
    
}
```

Do we see the issue here? We are repeating the specification of same functions foldl, map, flatmap for every datatypes. These functions are very generic that they can be implemented for variety of data-structures. How can we generalize them in Object oriented programming. Short answer we can't.

Generics is originally a concept in functional paradigm and object oriented paradigm borrowed this concept for implementing generic data-structures. (Remember data-structures are not objects). What we have here is some way to parameterize the data-structure (such as list, maybe, tree) in-addition to parameterizing the type of value the data-structure used to represent.

Since we cannot have such parameterization in JAVA, we cannot generalize these methods. This is serious limitation with JAVA in supporting functional programming. Having lamda (syntactic sugar for anonymous object) and functional interfaces (single behavior objects) doesn't help one to write functional code in JAVA.

Since we spoke about all functions being referentially transparent, reading and writing of values from/to the console which is a side effect should also be represented as pure functions. This is an issue since they are clearly side-effects and how to reason with them in referrentially transparent way?. Lets see IO and IOOps implementation to get it clear.

## Side-effect free Input and Output functions

```java

interface IO<T> {
    T unsafeIO();
}

interface IOOps {
    
    static<T, R> IO<R> flatMap (final Function<T, IO<R>> fn, final IO<T> io) {
        return () -> fn.apply(io.unsafeIO()).unsafeIO();
    }
    
    static <T> IO<Void> print (final T value) {
        return () -> {
            System.out.println(value);
            return null;
        };
    }
    
    static <T> IO<String> readString () {
        return () -> {
          Scanner in = new Scanner(System.in);
          return in.next();
        };
    }
    
    static <T> IO<Integer> readInt() {
        return () -> {
            Scanner in = new Scanner(System.in);
            return in.nextInt();
          };
    }
}

```
Lets consider the readString method. The method instead of returning the value read from console, returns the IO<String> object which reads a string value from the console in its unsafeIO() method. The call to unsafeIO() to retrieve the value can be postponed since the flatMap() method provided for IO<T> object can take the logic which depends on the value to be read.

How this helps to achieve referential transparency.?  Call to read() method returns a value which is same for every invocation with no side-effects. Any code which depends on the input string can be ingested via the IOOps.flatMap() method. Unless the unsafeIO method is called on the IO<T> object, nothing gets executed. 

The method unsafeIO is impure function but this needs to be invoked only once in the main method. Excluding this invocation, rest of the code is pure.

Lets see the consumer logic to better understand this.

```java

public static void main(String[] args) {
    Item[] items = new Item[] {
        new Item("Espresso", 120.00, 100),
        new Item("Cappuccino", 100, 100),
        new Item("Latte", 90, 100),
        new Item("Mocha", 140, 100),
        new Item("Cold Coffee", 75, 100)
    };
    
    IOOps.flatMap(__ ->
    IOOps.flatMap(
        name -> IOOps.flatMap(
        qty  -> {
            
            Tuple<List<Item>, List<Item>> orderedItems = CoffeeShop.order(List.of(items), name, qty);
            
            List<Function<Credit, Tuple<Credit, Either<PaidItem, UnPaidItem>>>> billedItems = ListOps.map(item -> Billing.bill(item), orderedItems._1);
            
            List<Tuple<Credit, Either<PaidItem, UnPaidItem>>> afterPayment = ListOps.apply(billedItems, List.of(new Credit(150)));
            
            Tuple<List<Credit>, List<Either<PaidItem, UnPaidItem>>> bankerAndPaidItems = TupleOps.fromList(afterPayment);
            
            List<UnPaidItem> unpaidItems = 
                ListOps.foldl(
                    (ups, epu) ->
                    MaybeOps.foldl(
                        (ups1, up) ->
                        List.cons((UnPaidItem)up, ups1),
                        ups,
                        EitherOps.foldRL(
                            (m, up) ->
                            Maybe.some(up),
                            Maybe.nothing(),
                            epu)),
                    List.<UnPaidItem>empty(),
                    bankerAndPaidItems._2);
            
            List<Item> store = CoffeeShop.shelve(orderedItems._2, unpaidItems);
            
            return IOOps.print(bankerAndPaidItems._1 + ", Paid / Unpaid = " + bankerAndPaidItems._2 +
                               "Store " + store);
        },
            IOOps.readInt()),                     
        IOOps.readString()),
    IOOps.print("Welcome to Coffee Shop\nPlease enter name and quantity?")).unsafeIO();
}
```

We are composing multiple IOOps functions here to perform input and output operations. The readString function returning IO<String> object can take the logic which depends on input string read from console without actually performing the read operation. Thus IO<String>::flatMap is pure function. We can compose any number of IO functions and logic depending on input and output using the flatMap and the code is still pure. 

The actual input and output operations are performed only when we call unsafeIO() function. This call can be postponed thanks to function composition. The unsafeIO() needs to be called only once and all the effects are applied in the order in which the code was composed.

Our use-case starts with placing an order by specifying the name and quantity to order function which returns the ordered items list and updated store. Next by mapping each ordered item using bill method, we get list of functions taking credit returning updated credit and either paid or unpaid item.

We need to apply the functions to credit object and get the results in another list. Now we need to extract the paid and unpaid items. For all unpaid items, we need to call shelve to return it back to the store. The output can be printed to console using IOOps.print() function.

Now if we look at the main method, the code is very verbose. Its because JAVA language is very verbose for functional programming and more than that it lacks the generalization capabilities to avoid code repetition. The same code written in functional languages like haskell would be much more concise and clear. 

One of the good things about this code is the fold method which fuels most of the functional APIs. It can be used in object oriented programming as well for iteration. Any imperative code involving for loops on collections can be represented using fold operation. 

JAVA streams has reduce API which is nothing but a fold method. Any imperative code involving for-loop on collections can be refactored using this reduce method.
For Example the order function we saw earlier in this blog can be written using reduce API of stream as follows:

```java

Tuple<ArrayList<Item>, ArrayList<Item>> orderedItems = 
        Arrays.asList(items).stream()
        .reduce(
            new Tuple<>(new ArrayList<Item>(), new ArrayList<Item>()),
            (tis, item) -> {
                if (item.qty > qty && item.name.equalsIgnoreCase(name)) {
                    ArrayList<Item> left = new ArrayList<>(tis._1);
                    ArrayList<Item> right = new ArrayList<>(tis._2);

                    left.add (new Item (item.name, item.price, qty));
                    right.add (new Item (item.name, item.price, item.qty - qty));

                    return new Tuple<>(left, right);
                } else {
                    ArrayList<Item> right= new ArrayList<>(tis._2);
                     right.add(item);
                     return new Tuple<>(tis._1, right);                    
                }
        }, 
        (l1, l2) -> l2);
```
This is clearly an improvement over imperative coding using for loop and state mutation. But the better way to code in Java is to define objects encapsulating state and expose well defined behaviour.

# Object oriented Implementation

```java

interface Either<A, B> {
    Either <A, B> ifSuccess (final Consumer<A> action);
    Either <A, B> ifFailure (final Consumer<B> action);
}

interface Inventory {
    Order order (final String name, final int quantity);
    void shelve (final String name, final int quantity);
}

interface Order {
    Either<PaidItem, UnPaidItem> bill (final Credit credit);
}

interface Credit {
    boolean pay (final double price);
}

interface PaidItem {
    void checkOut();
}

interface UnPaidItem {
    void revert (final Inventory inv);
}
```

And the Consumer logic is ...

```java
inventory.order("Mocha", 1).bill(credit)
.ifSuccess (PaidItem::checkOut)
.ifFailure(up -> up.revert(inventory));
       
```
We can understand the code very easily by looking at the object interaction without paying attention to the implementation details. 

Functional programming is easier in languages where First Class Functions, TypeClass parameterization, Lazy evaluation, pattern matching, syntactic sugar for chaining map and flatMap APIs (Example Haskell do Notation, for comprehension in scala), Higher kinded types and many other functional features are present. In all other languages (like CPP and JAVA) its just an overkill. 

In this post we might have observed that functional programming revolves around values. There is no concept of encapsulation. They depend on pure functions and values instead of dealing with variables and state mutation to solve the problem.

Pure object oriented code is both concise and easy to understand and any complexity with respect to state mutation is confined within the premise of an object (Thanks to encapsulation). Object oriented programming confine the variables and effects within the premise of an object whereas functional programming eliminates them (refer IOOps). Both address the complexity due to variables and effects but follows different approach. 

Intermixing both the styles in single application is an abomination and it will violate the principles of one paradigm when we choose another. In this post we broke encapsulation in every class we have defined to adopt functional style. Pure object oriented code doesn’t return values and functional programming is all about coding around values. 

There are definitely many ideas we can borrow from functional world but should implement them in pure object oriented fashion to use in JAVA. Examples are Maybe for 
null handling, Either for exception handling, fold for iteration, and so on.

If anyone has any comments about the blog contents, please feel free to comment below.