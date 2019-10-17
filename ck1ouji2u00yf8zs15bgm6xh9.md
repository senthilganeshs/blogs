## Effects, Transformations and Values

An application written using Object Oriented concepts is often hard to read and understand. It's not because of the presence of too many classes and objects but due to leaky objects which introduces lot of procedural coding in the consumer logic.

Object is leaky if it exposes its internal state either via public fields or via accessors and mutators. Objects which returns primitive values or data-structures for any behavior also causes the consumer logic to have procedural code. 

Avoiding such APIs will help keep the consumer logic more concise and easy to understand. 

Behavior of an object can be classified as Effect or Transformation. Effect methods can have side-effects. The side-effect may involve mutating its internal state. They can have no return type or can return themselves. Transformation functions should avoid side-effects and should construct new objects and return. They should not return primitive values or data-structures.

Whenever we have an existing object which has an API that returns values or data structures, we need to look into consumer logic and come-up with new type(s) which exposes the behavior which the consumer can directly use. The APIs can be refactored to return this new type instead of the value.

Some times, refactoring is costly and we would want to re-use the existing code as is but also want to keep the consumer logic concise. In those cases we can define new types with only the behavior the application needs and hide the existing objects within the implementation of those new objects.

Lets see a simple pattern matching example in java.

```java
final String text = "aabbabcaabbabcabcabc";
Matcher matcher = Pattern.compile("(a|b)+").matcher(text);
int count = 0;
while (matcher.find()) {
    System.out.println(
        text.substring(
            matcher.start(), 
            matcher.end()));
    count ++;
}
``` 
Based on our previous description, lets classify the methods used in the above code.

- "Pattern.compile" - factory method which creates a Pattern object.
- "pattern.matcher" - is a transformation method which creates a Matcher object. 
- "*matcher.find()*" returns boolean which is a primitive value.
- "*matcher.start()*" , "*matcher.end*" returns integer indexes into the original string.

The Matcher object is leaky since it returns the state of the object via API's such as start, end, and returns primitive values for find API. Using the library as-is will cause the consumer logic become complex.

Since these API's are from JAVA standard library, refactoring is not an option. But we can define new types and depend only on them in the consumer logic. The objects implementing the new type can use the java.util.regex.Matcher objects for its implementation. 

```java
interface Matches {
    void consume (final Consumer<String> action);
    <R> R consume(
                    final R identity, 
                    final BiFunction<String, R, R> accum, 
                    final BiFunction<R,R,R> combiner);
}
```

Since pattern matching is still a very generic use-case, we wanted to support both effects and transformation logics to consume the matched strings.

```java
interface Matches {            
    void consume (final Consumer<String> action);            
    <R> R consume (
            final R start, 
            final BiFunction<String, R, R> accum, 
            final BiFunction<R,R,R> combiner);
                        
    public static Matches create (
                             final Matcher m, 
                             final String text) {
        return new TextMatches (m, text);
    }

    final static class TextMatches implements Matches {
        private Matcher orig;
        private final String text;
               
        TextMatches (final Matcher m, final String text) {
            this.orig = m;
            this.text = text;
        }

        @Override
        public void consume(final Consumer<String> action) {
            while (orig.find()) {
                action.accept(matchedText());                        
            }
            orig = orig.reset();//reset state
        }

        private String matchedText() {
            return text.substring(orig.start(), orig.end());
         }

        @Override
        public <R> R apply(
            final R start, 
            final BiFunction<String, R, R> accum,
            final BiFunction<R, R, R> combiner) {
                  
            R state= start;
            while (orig.find()) {
                state = combiner.apply(
                               accum.apply(
                                 matchedText(), state), state);
            }
            orig = orig.reset(); //reset state.
            return state;
        }
    }
}
```

Now with these wrappers the consumer logic will become

```java
Matches.create(
    Pattern.compile ("(a|b)+").matcher (text), 
    text)
.consume(System.out::println);

//to count the total number of matches.
Matches.create (
    Pattern.compile ("(a|b)+").matcher (text),
    text)
.consume(0, (s, n) -> n + 1, (a, b) -> a);
```

Now in our consumer logic, lets analyze the methods.
- "consume" - effect method takes the logic which depends on matched string.
- "consume" - transformation method which takes the start value, accumulator logic (logic to add new matched string to the existing result) and combiner logic (logic to reduce two results to one).

There are no methods returning values or data-structures and hence the consumer logic is concise and easy to read and understand.

We have seen how even existing code having APIs' which return the values or data-structures can add to complexity in the consumer logic and how we can avoid them by introducing additional types which doesn't violate encapsulation.