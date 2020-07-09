# Actor Model

One of Alox's main pillars is the use of the Actor Model.
Actors can hold data like classes or structs, but they also have functions
and behaviours associated with them.

```rust
actor A {
    behave ping(n: Int32, b: &B) {
        b.pong(n, &this)
    }
}

actor B {
    behave pong(n: Int32, a: &A) {
        a.ping(n, &this)
    }
}
```

Calling a behaviour is different from calling a normal function. Instead of
the body being executed immediately, it is scheduled to run. The runtime
manages the scheduling and execution of behaviours. These behaviours have
some interesting characteristics:

* references are not allowed as arguments except for references to actors*
    * If I pass a reference to a struct that’s in the heap or stack on my
    machine to an actor running on a different machine, my reference is
    going to point to garbage! The only references that are safe to go over
    the wire are references to other actors. All actors have an ID that can
    be serialized.
* behaviours have no return type (or the return type will be required to be a Future if those are implemented)
* to execute a behaviour, you need to send a _message_ to the _mailman_

*: actors on the same _stage_ can send references of memory to each other


## Messages

Messages contain three parts: the id of the actor it’s for, the name of the
behaviour to run, and the serialized argumets.

Let’s look at a more complex example.

```rust
actor A {
    behave ping(n: Int32, b: &B) {
        b.pong(n, &this)
    }
}

actor B {
    behave pong(n: Int32, a: &A) {
        a.ping(n, &this)
    }
}

actor Main {
    behave main() {
        let n = 2
        let a = new A()
        let b = new B()
        a.ping(n, &b)
    }
}
```

`Main.main` will create two new actors and start off the pinging and ponging.
`a.ping(n, &b)` will queue a mesage to be sent that will send whatever the value
of `n` is and the ID of the `b` actor. Every actor has an ID that is created by 
the scheduler.

## The Scheduler, Worker Threads, & Mailmen



_The Scheduler_ is the process that controls where everything is. When you start
an Alox instance, you start The Scheduler. Each scheduler is its own universe that
communicates with other schedulers. You can think of it as the process you see
when you run `top`. You can run multiple schedulers on one device, but the proper
usage would be to run them on different devices. They communicate over some 
protocol I haven’t designed yet going probably over TCP.

Schedulers spawn two important things: _worker threads_ and _mailmen_. Mailmen put
messages in the mailbox, and workers read the messages and execute the behaviours
Mailboxes act as the middle men. They’re generic multi-consumer multi-producer
(MCMP) queues. Mailmen run on their own threads, there will probably be a few per
scheduler, and every call to a behaviour will be a direct call to a mailman.
Mailmen are also responsible for sending messages across the wire. If the ID in
the message refers to an actor running under a different scheduler, the mailman is
responsible for passing the message through his own scheduler to a mailman on the
correct scheduler.

Worker threads consume a message from the mailbox and run the designated behaviour
on the actor it refers to. This will run the AOT compiled code corresponding to
the function with a pointer to the actor structure.

For example, `a.ping(n, &this)` will compile to instructions that tell the Mailman
to send a message to the mailbox. Then, the worker thread will read the message
and run the generated function `A_ping(&a, b, &this)`.
