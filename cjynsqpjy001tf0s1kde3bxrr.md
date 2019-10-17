## Remote Objects .. The Object Oriented Way

In Object oriented programming, we usually define objects with behavior encapsulating (hiding) the state and the objects "talk" to each other to get any "work" done. This is in-contrast with procedural programming where only behavior is divided in to smaller sub-problems. There usually is less control on managing the state.

In Object oriented programming, any method invocation on an object is referred as "message passing". 

res = foo.action(bar);

The message "action" is sent to object "foo" with arguments "bar" to get the result "res". Since the objects are all available in the same thread all these details are concisely hidden using the "." notation.

Consider if implementations of foo is not available for the caller who wanted to compute res but he only has bar. May be the object foo is available in a server thread (or process) and input arguments ("bar") to send the message "action" to object "foo" is available in another thread(or process). How we can work with these scattered objects.?

Let us expand our "." notation in message passing further with custom types.


```
interface Message<T> {
    void execute (final T foo);
}

interface Foo {
    void action (final Bar bar);
}

interface Mailer<T> {
    void send (final Message<T> msg);
}

interface Mailbox<T> {
    void receive (final Consumer<Message<T>> action);
}
``` 

Consider above interfaces substituting the "." operator we have in our message passing syntax.

res *{Mailbox::receive}* = foo.*{Mailer::send}*action*{Message<Foo>}*(bar);

The caller would want to receive a message from its mailbox by sending a message "action" via the Mailer. On the other end, the message is reached to the other party executes the message by supplying "foo". The message in-turn encapsulates the input arguments (bar) which it supplies to foo and invoke the action.

```
BlockingQueue<Message<Foo>> fooQueue = new ArrayBlockingQueue<>(1);
CompletableFuture<Void> server = CompletableFuture.runAsync(() -> {
    Foo foo = Foo.create();
    Mailbox<Foo> mailbox = Mailbox.local(fooQueue);
    mailbox.receive(msg -> msg.execute(foo));
});

CompletableFuture<Void> client = CompletableFuture.runAsync(() -> {
    Bar bar = Bar.create();
    Mailer<Foo> mailer = Mailer.local(fooQueue);
    mailer.send(foo -> foo.action(bar));
});
``` 
In the above example we have abstracted out the infrastructure required to perform "message passing".  By changing the implementations of Mailer and Mailbox, we can support remote objects coming in from any environment.

For example, to extend the above example to TCP / IP only requires implementation of new Mailbox and Mailer implementation to use the TCP socket and encode / decode the message over the network. The rest of the business logic remains unaffected. Porting the code to any other infrastructure in the future is also easy.

We might have seen several examples of Socket programming using the procedural constructs which doesn't scale beyond the given problem statement. On contrast, designing in object oriented style allows extending even a small program (like this example) to something very big without altering the consumer logic.