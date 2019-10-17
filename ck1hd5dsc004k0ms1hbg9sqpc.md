## Functional and Object Oriented Programming

Program execution is non deterministic. The program instructions and memory access does not follow program order. Access to variable which is a place holder in memory for storing values (state) is also non-deterministic. Variables introduce time as a factor in code and due to non-deterministic model of program execution, we need to place special memory fencing instructions and locks to ensure consistent reads and writes of variables by multiple threads. 

Locks introduce further complexity that when there is hierarchy of locks guarding a variable, the code will deadlock if the order of locks is different elsewhere. 

Program instructions are executed by processor but the instructions accessing hardware such as printing to the console, reading from / writing to files, sending data via network, is delegated to other hardware devices. So, processor needs to context switch the execution whenever access to the other hardware is needed or time slice for a given program execution has expired. 

All this factor contributes to the complexity of writing new code or understanding existing code. 

The rest of the blog post discusses how the functional and object-oriented programming paradigms address this complexity.

Since using variables to represent state is the root cause for complexity in coding, both the paradigms take the variables out of the equation.

## Functional Programming
Functions are first class objects in this paradigm. Composing functions will get the work done. State is also represented as a function. Clear separation between side-effecting code (code interacting with hardware, mutating state, and so on) and the pure functions(function with no side effects  where function calls can be replaced with result of the function). 

Code is deterministic if it involves only function composition. Since the functions are pure there is no notion of time playing a part. The program execution is still non-deterministic, but the logic is defined in deterministic way and hence the code is easy to read and understand.

## Object Oriented programming.
Objects are first class entities in this paradigm. Objects encapsulate state (hides) and expose only behavior. Objects can be composed or interact with each other to get things done. 

Code dealing with only objects is deterministic (like functional paradigm). There is no distinction between pure and impure functions. Since the state is hidden for outside access, the complexity in dealing with state is confined within the premises of a single object. The more granular the object is, lesser the complexity one must deal with.

Let's see an example of simple number game to illustrate how the two paradigm go about solving the problem. This example is taken from  [here](https://wiki.haskell.org/State_Monad) .


```
module StateGame where

import Control.Monad.State

-- Example use of State monad
-- Passes a string of dictionary {a,b,c}
-- Game is to produce a number from the string.
-- By default the game is off, a C toggles the
-- game on and off. A 'a' gives +1 and a b gives -1.
-- E.g 
-- 'ab'    = 0
-- 'ca'    = 1
-- 'cabca' = 0
-- State = game is on or off & current score
--       = (Bool, Int)

type GameValue = Int
type GameState = (Bool, Int)

playGame :: String -> State GameState GameValue
playGame []     = do
    (_, score) <- get
    return score

playGame (x:xs) = do
    (on, score) <- get
    case x of
         'a' | on -> put (on, score + 1)
         'b' | on -> put (on, score - 1)
         'c'      -> put (not on, score)
         _        -> put (on, score)
    playGame xs

startState = (False, 0)

main = print $ evalState (playGame "abcaaacbbcabbab") startState
``` 

If we consider the main function, it has functions which work together(compose) to get the work done. 

The object oriented implementation of the above problem is as follows:


```
interface Game {
    State play (final String input);
}

interface State {
    State consume (final BiFunction<Boolean, Integer, State> fn);
}

interface Transition {
    State accept (final Character ch);
}

public static void main (final String [] args) {
    State.print(Game.numberGame (State.create (false, 0)).play("abcaaacbbcabbab"));
}

``` 
In Comparison with the  functional code, 

- The states toggle and score are hidden inside the State object. 
- The case expression is implemented using Transition object. 
- The initial state of the game is again a state which is encapsulated by the Game object. 
- Since the score is encapsulated within State object we need another implementation of State which can print the score.

If we see the main method above, its declarative in nature. There are no variables used and state is encapsulated by different objects and only behavior is exposed from each of them. This code is easy to understand and there is no notion of time involved in understanding the code.

```
final static class NumberGame implements Game {
    NumberGame (final State start) {
        this.start = start;        
    }
    @Override
    public State play (final String input) {
        return input.chars()
            .mapToObj(i -> (char) i)
            .reduce(
                start,
                (s, ch) -> Transition.create(s).accept(ch),
                (s, t) -> t);        
    }
}

final static class NumberState implements State {
    NumberState (final boolean toggle, final int score) {
        this.toggle = toggle;
        this.score = score;
    }
    @Override
    public State consume (final BiFunction<Boolean, Integer, State> fn) {
        return fn.apply(toggle, score);
    }
}
//returns same state after consume but print the score
public static State print (final State state ) {
    return state.consume((t, s) -> printToConsole(s, state));
}

private static State printToConsole (final int score, final State state) {
    System.out.println (score);
    return state;
}

final static class NumberATransition implements Transition {
    NumberATransition (final State start, final Transition next) {
        this.state= start;
        this.next = next;
    }
    @Override
    public State accept (final Character ch) {
        return (ch == 'a') ? state.consume((t, s)-> State.create(t, (t) ? s+1 : s)) : other.accept(ch);
    }
}

/*
* Similar implementations for NumberBTransition, NumberCTransition, NumberXTransition
*/

public static Transition create (final State start) {
    return 
        new NumberATransition(start,
            new NumberBTransition(start,
                new NumberCTransition(start,
                    new NumberXTransition(start))));
}
``` 

We have seen how an imperative problem such as this one can be written using functional and object oriented concepts and we have seen how the code (main function) becomes easy to read and understand following the paradigms strictly. 

The example here doesn't involve sharing of state across multiple threads. But even for a problem involving multiple threads sharing the states, the object oriented implementation hides the states and locks guarding the states within the implementation of the object. Only objects are shared across multiple threads. The main function which has only objects interacting with each other remains the same.