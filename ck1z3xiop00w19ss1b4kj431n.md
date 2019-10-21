## Array Operations and Recurrence in Object Oriented Programming

Arrays are fundamental data structures. They represent collection of ordered items. Large set of problems can be represented using Arrays and variety of algorithms exists to solve them. Arrays are recursive objects. Problems represented using Arrays can be solved by performing operations such as iteration, generation of subsets / subarrays / permutations on the array elements.

Lets see what each operation means for an array {a,b,c} 
- iteration, visits each element of the array in the same order {a,b,c}
- subsets, generates the following subsets and visits them {{a,b,c}, {a,b}, {a,c}, {a}, {b,c}, {b}, {c}, {}} 
- subarray, generates the following subsets and visits them {{a}, {a,b}, {a,b,c}, {b}, {b,c}, {c}}
- permutations, generates the following subsets and visits them {{a,b,c}, {a,c,b}, {b,a,c},{b,c,a}, {c,a,b},{c,b,a}}


Object oriented implementation of array should provide API's to perform the above operations so that the same boilerplate code is not repeated for solving different problems.

Lets start by defining an interface for Array as follows:

```java
interface Array<T> {
    <R> R iterate (final R start, final BiFunction<T, R, R> accum, final BiFunction<R,R,R> combiner);
    default <R> R iterate (final R start, final BiFunction<T,R,R> accum) {
        return iterate (start, accum, (r1, r2) -> r2);
    } 

    void forEach (final BiConsumer<Integer, T> action);

    <R> Array<R> subsets (final R start, final BiFunction<T,R,R> accum, final BiFunction<R,R,R> combiner);

    default <R> Array<R> subsets (final R start, final BiFunction<T,R,R> accum) {
        return subsets (start, accum, (r1, r2) -> r2);
    }

    <R> Array<R> permute (final R start, final BiFunction<T,R,R> accum, final BiFunction<R,R,R> combiner);

    default <R> Array<R> permute (final R start, final BiFunction<T,R,R> accum) {
        return permute (start, accum, (r1, r2) -> r2);
    }

    <R> Array<R> subarrays (final R start, final BiFunction<T,R,R> accum, final BiFunction<R,R,R> combiner);
    default <R> Array<R> subarrays (final R start, final BiFunction<T,R,R> accum) {
        return subarrays (start, accum, (r1, r2) -> r2);
    }
}
```
We will get to the implementation of these API's later at the end of the post but will see the consumer logic first. In Object oriented programming, we hide the complexity of code within some object and aim to have more concise code for the consumer. 

By repeatedly applying the above logic everywhere, we will end-up with more fine grained objects and the code becomes much more readable.

For the sake of this blog we will restrict to have concise code only for the first level of consumer who wanted to solve some well known procedural problems using Arrays!


```java

//Finding largest subset sum.
final TwoInt zero = new TwoInt(0, 0);

System.out.printf ("Largest subset sum = %s\n",
    Array.of(new Integer[] {2,3,5,7,10,15})
    .subsets (zero, 
        // a = size of subset, b = sum of subset elements
        (si, t) -> new TwoInt (t.a + 1, t.b + si))
    .iterate (zero,
        //select subsets whose sum equals 'sum'
        (t, r)  -> (t.b == sum) ? t : r,
        //select subset which has more elements
        (t1, t2) -> (t1.a > t2.a) ? t1 : t2));
```
Procedural implementation for the above problem is  [here](https://www.geeksforgeeks.org/maximum-size-subset-given-sum/) 

Lets see another problem where we find smallest contiguous subarray sum whose sum is greater than or equal to 'S'

```java

System.out.printf("Smallest subarray sum greater than %d = %s", 
7,
Array.of(new Integer[]{2, 1, 5, 2, 3, 2})
.subarrays(
    zero,
    (si, t) -> new TwoInt(t.a + 1, t.b + si)) //add index and sum to twoint
.iterate (
    zero,
    (t, r) -> (t.b >= 7) ? t : r,  //select sum greater than 7
    (t1, t2) -> (t1 != zero && t1.a < t2.a ) ? t1 : t2)); //select smallest subarray

```

If we observe the problems, both are finding sum of elements in the subset and picks the one with large or small number of elements in it. The key difference between these two problems is in the problem space they operate on. The first problem needs to evaluate all possible subsets which is 2 ^ N whereas for the second problem its N * (N + 1) / 2. 

The complexity of problem space is hidden using the apis subsets, and subarrays. The logic to enumerate the subarrays, finding the intermediate result (computing the sum and number of elements) and selecting the optimal solution (smallest subarray) is expressed declaratively.

Now lets see how we can express recurrence with object oriented API's. Any recurrence should have a tail case and recurrence. Lets start writing interfaces for those.

```
interface Base<I, R> {
    Optional <R> f (final I input);
}

interface Recurrence <I, R> {
    R f (final I input, final Recurrence <I, R> self);


    static <I, R> R create (
                              final I input, 
                              final Recurrence<I, R> recur, 
                              final Base<I, R>...bases) {
        final Base<I, R> base = i -> {
            for (final Base<I, R> b : bases) {
                if (b.f(i).isPresent()) 
                    return b.f(i);
            }
            return Optional.empty();
        };
        final Recurrence<I, R> withBaseCases = 
        (i, self) -> base.f(i).orElseGet(() -> recur.f(i, self));

         return withBaseCases.f (input, withBaseCases).get();
    }
}


//Program to find number of different ways a change is made for sum 'S' using denominations denom[]

final int denom[] = {2, 5, 3, 6};
System.out.printf ("coin change for %d using denominations %s = %s",
    10, Arrays.toString(denom),
     Array.recur(
         new TwoInt (denom.length, 10), 
         //Recurrence 
         (t, self) -> self.f (new TwoInt (t.a - 1, t.b), self) + //exclude denom
                      self.f (new TwoInt (t.a, t.b - denom[t.a - 1]), self), //include denom
        //base cases
        // (1) sum =0, change can be made
         t   -> (t.b == 0) ? Optional.of(1) : Optional.empty(), 
         // (2) sum < 0, no change can be made.
         t   -> (t.b < 0)   ? Optional.of(0) : Optional.empty(), 
         // (3) no denominations available and sum > 1, no change is possible.
         t   -> (t.a <= 0 && t.b >= 1) ? Optional.of(0) : Optional.empty()));
```

The above logic is very clear in expressing what we wanted to do to solve this coin-change problem. The recurrence step specifies clearly that the total number of ways to get change for sum 's' using denominations denom[] is sum of the total number of ways in which the ith denomination was excluded and included. When denomination is included, the sum is subtracted by the denomination value in the recurrence. The recurrence cases are followed by the base cases. The base cases are self explanatory. 

This abstraction helps consumers to concentrate only on the recurrence logic needed to solve the problem rather than on the boilerplate code needed to implement the recurrence itself.

There is one issue with the above code. It computes the same sub-problems repeatedly. Efficient algorithm is to memoize the result of sub problems and re-use it. Lets see how we can solve the memoization without introducing any clutter.

```java

public static <I, R> Recurrence <I, R> memoized (
    final Function<I, Integer> indexFn,
    final Recurrence <I, R> other) {
    return new Memoized <> (indexFn, other);
}

final static class Memoized <I, R> implements Recurrence <I, R> {
    
    private final Recurrence <I, R> other;
    private Object[] table;
    private final Function <I, Integer> indexFn;

    Memoized (
        final Function<I, Integer> indexFn, 
        final Recurrence <I, R> other) {
 
       this.indexFn = indexFn;
       this.other = other;
       this.table = null;
    }

    @Override
    public R f (final I input, final Recurrence <I, R> self) {
        final int n = indexFn.apply (input);
        if (table == null) {
            table = new Object [n + 1];
        }
        if (table [n] == null) {
            table [n] = other.f (input, self);
        }
        return (R) table [n];
    }
}

final int denom[] = {2, 5, 3, 6};
System.out.printf ("coin change for %d using denominations %s = %s",
    10,
     Arrays.toString(denom),
     Array.recur(
         new TwoInt (denom.length, 10), 
         //Recurrence 
         Recurrence.memoized (
             t -> t.a * 10 + t.b,  //return index into memoization table
             (t, self) -> self.f (new TwoInt (t.a - 1, t.b), self) + //exclude denom
                          self.f (new TwoInt (t.a, t.b - denom[t.a - 1]), self)), //include denom
        //base cases
        // (1) sum =0, change can be made
         t   -> (t.b == 0) ? Optional.of(1) : Optional.empty(), 
         // (2) sum < 0, no change can be made.
         t   -> (t.b < 0)   ? Optional.of(0) : Optional.empty(), 
         // (3) no denominations available and sum > 1, no change is possible.
         t   -> (t.a <= 0 && t.b >= 1) ? Optional.of(0) : Optional.empty()));

```
 
If we compare the above code with the previous snippet, the only difference is that the recurrence object is now wrapped within the Recurrence.memoized object. The memoized object uses table to cache the sub-problem results and speeds up the recurrence.

Object oriented programming concepts can help avoid procedural coding even for solving procedural problems like the ones we have seen in this blog. This API abstraction will help consumer to only concentrate on logic to solve the problem without needing to think about the boilerplate code needed to implement the logic.

As promised, below is the implementation of the Array interface

```java
final static class OneDimension<T> implements Array<T> {
    
    private final T[] arr;

    OneDimension(final T[] arr) {
        this.arr = arr;
    }
    
    @Override
    public <R> Array<R> permute(final R start, final BiFunction<T, R, R> accum,
        final BiFunction<R, R, R> comb) {
        
        List<R> list = new ArrayList<>();
        
        T[] result = newArray(arr);
        
        comb(
            arr.length, 
            0, 
            arr, 
            result, 
            res -> list.add(
                Array.of(res).iterate(start, accum, comb)));
        
        return Array.of((R[])list.toArray());
    }
    
    
    private void comb (final int n, final int r, final T[] arr, final T[] result, final Consumer<T[]> action) {
        if (n == r) {
            action.accept(result);
            return;
        }
        
        for (int i = 0; i < n ; i ++) {
            int j = 0;
            for (; j < r ; j ++) {
                if (result[j] == arr[i])
                    break;
            }
            if (j < r)
                continue;
            result[r] = arr[i];
            comb (n, r + 1, arr, result, action);
        }
    }
    
    @Override
    public <R> Array<R> subarrays (final R start, final BiFunction<T, R, R> accum,
        final BiFunction<R, R, R> comb) {
        
        final List<R> list = new ArrayList<>();
        
        subarray (
            arr,
            res -> list.add(
                new OneDimension<>(res).iterate(start, accum, comb)));
        
        return new OneDimension<R>((R[]) list.toArray());
    }
    
    
    private void subarray (final T[] arr, final Consumer<T[]> action) {
        for (int i = 0; i < arr.length; i ++) {
            for (int j = i; j < arr.length; j ++) {
                final T[] res = newArray(arr);
                for (int k = i, l = 0; k <= j; k ++, l ++) {
                    res[l] = arr[k];
                }
                action.accept(newArray(res, true));
            }
        }
    }

    @Override
    public <R> R iterate(final R start, final BiFunction<T, R, R> accum, final BiFunction<R, R, R> comb) {
        
        if (arr == null || arr.length == 0)
            return start;
        
        final List<R> results = new ArrayList<>();
        results.add(start);
        
        R result = start;
        for (final T a : arr) {
            result = accum.apply(a, result);
            results.add(result);
        }
        
        result = results.get(0);
        
        for (int i = 1; i < results.size(); i ++) {
            result = comb.apply(result, results.get(i));
        }
        
        return result;
    }

    @Override
    public void forEach(final BiConsumer<Integer, T> action) {
        int i = 0;
        for (final T a : arr) {
            action.accept(i, a);
            i ++;
        }
    }

    private <R> void  subseq (final int r, final int c, final int n, final T[] arr, final T[] result, final Consumer<T[]> action) {
        if (r == n) {
            action.accept(newArray(result, true));
            return;
        }
        result[c] = arr[r];
        subseq (r + 1, c + 1, n, arr, result, action);
        result[c] = null;
        subseq(r + 1, c, n, arr, result, action);                
    }

    @Override
    public <R> Array<R> subsets(final R start, final BiFunction<T, R, R> accum,
        final BiFunction<R, R, R> comb) {
        
        final List<R> lst = new ArrayList<>();
        
        subseq (
            0, 
            0,
            arr.length, 
            arr, 
            newArray(arr), 
            res -> lst.add(new OneDimension<>(res)
                    .iterate(start, accum, comb)));                       
        return new OneDimension((T [])lst.toArray());
    }
    
    private T[] newArray(final T[] arr) {
        final T[] result = Arrays.copyOf(arr, arr.length);
        for (int i = 0; i < result.length; i ++) {
            result[i] = null;
        }
        return result;
    }
    
    private T[] newArray(final T[] arr, final boolean notNull) {
        if (notNull) {
            int i = 0;
            for (; i < arr.length; i ++) {
                if (arr[i] == null) {
                    break;
                }
            }
            return Arrays.copyOf(arr, i);                    
        } else {
            return newArray(arr);
        }
    }
}

```