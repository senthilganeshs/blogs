## Exception Handling... Object oriented way

Exceptions are pretty common in Java and is still considered cleaner way of indicating failure to the caller. The caller should decide whether to act on the failure or should delegate the exception handling to his caller. 

The issue with try.. catch blocks in Java is that they are not objects and its not possible to compose code which throws different exceptions. Nested try..catch blocks offers poor readability. Any exceptions not handled in the catch block would result in application going into inconsistent state.

In Functional Programming, we have a data type called Either<A,B> which defines value present in the type is either of type A or of type B. Substituting value and exception for A and B, we can compose code which depends on value and code which handles / re-throws other exceptions separately. The Either<A,B> type is an extension of Optional<A> or Maybe<A> types which defines a value if present is of type A and if its not present is empty. 

We can take the same idea into the object oriented world as well by defining the following type:

```
interface Either <A, E extends Exception> {
    Either <A, E> ifSuccess (final Consumer<A> action) ;
    Either <A, E> ifFailure (final Consumer<E> action);

    default <R, Y extends Exception> Either<R, Y> andThen (final Function<R, A, Y> fn, final java.util.function.Function<E, Y> errorFn) {
        final AtomicReference<Either<R, Y>> result = new AtomicReference<>();
        ifSuccess(t -> {
            try {
                  final R value = fn.code(t);
                   result.set(new Success<>(value));
            } catch (final Exception e) {
                   result.set(new Failure<>((Y) e));
             }
        }).ifFailure (oe -> result.set(new Failure<>(errorFn.apply(oe))));
        return result.get();
    }
}

interface Supplier<A, E extends Exception> {
    A code() throws E;
}

interface Function<A, R, E extends Exception> {
    A code(final R value) throws E;
}

public static <T, X extends Exception> Either <T, X> wrap (final Supplier<T, X> th) {
    try {
        final T value = th.code();
        return new Success<>(value);
    } catch (Exception excep) {
        return new Failure<>((X) excep);
    }
}

final static class Failure <A, E extends Exception> implements Either<A, E> {
    private final E excep;
    Failure (final E excep) {
        this.excep = excep;
    }

    @Override
    public Either<A, E> ifSuccess(final Consumer<A> action) {
        return this;
    }

    @Override
    public Either<A, E> ifFailure (final Consumer<E> action) {
        action.accept(excep);
        return this;
    }
}

final static class Success <A, E extends Exception> implements Either<A, E> {
    private final A value;
    Success(final A value) {
        this.value = value;
    }

    @Override
    public Either<A, E> ifSuccess(final Consumer<A> action) {
        action.accept(value);
        return this;
    }

    @Override
    public Either<A, E> ifFailure (final Consumer<E> action) {
        return this;
    }
}


public void testEither() throws Exception {
    Either
    .wrap (EitherTest::only10)
    .andThen(EitherTest::onlyTen,
                    e -> new IOException(e))
    .andThen(EitherTest::expectEleven,
                    e -> new RuntimeException(e))
    .ifSuccess (System.out::println)
    .ifFailure (System.err::println);
}

private static String expectEleven(final String value) throws RuntimeException {
    if (value == "Eleven") return "Success";
    throw new RuntimeException ("Not Eleven");
}

private static int only10 () {
    return 10;
}

private static String onlyTen (final int x) throws IOException {
    if (x == 10) return "Ten";
    throw new IOException("Not 10");
}

``` 

The test code above shows how different code which throws different exceptions can be composed together and how the exception handling can be delayed until we are ready to handle it. The Either object can be shared freely and expect the caller to compose code which depends on the value all along. An exception occurred at some point will simply not execute the composed code. When we are ready to handle the exception we can call ifFailure method specifying what to do with the exception.

