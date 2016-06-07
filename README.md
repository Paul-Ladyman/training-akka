Fast Track to Akka with Scala
-----------------

*General Tips*

* [Akka Docs](http://doc.akka.io/docs/akka/2.4.7/scala.html), [API](http://doc.akka.io/api/akka/2.4.7/?_ga=1.263547788.1604209363.1464341691#package), [letitcrash.com](http://letitcrash.com/)

* Using a Seq in Scala can be problematic because the compiler may infer that the Seq should be mutable depending on how its used! This may have unforeseen consequences. Its generally best to use a concrete implementation (Seq is abstract) that is mutable or immutable as desired.

* Actors can be named but this is optional - the Actor System will give it a name if one is not provided

* To get the human-readable part of an auto-generated Actor name you can use `actorRef.path.name`

* In order to log unhandled messages, set the configuration setting `akka.actor.debug.unhandled` to on

* Akka uses the [Typesafe Config Library](https://github.com/typesafehub/config), [Akka Config Docs](http://doc.akka.io/docs/akka/current/general/configuration.html).

Akka
====

* Claims to offer simple concurrency, distribution (across networks) and fault*tolerance
** By abstracting away concurrency primitives on the JVM (threads, locking, etc.)
* on the JVM
* Reactive, message driven

What is a Reactive app?
=================

* Responsive in a timely manner
* Stays responsive in the face of failure (resiliency)
* Stays responsive in the face of high load (elastic). May also free up resources during quiet periods (scales down)
* Message driven

Simpler concurrency
================

* Actors look like single-threaded code but are in fact concurrent
* No locks or other primitives needed

Simpler distribution
==================

* Distributed by default, even locally

Simpler fault tolerance
================

* Communication decouples from failure handling
* Failures are handled by Supervisors, not callers of a particular actor

Actor model
===========

* The actor model was invented in 1973 by Carl Hewitt (it is not an Akka-specific concept)
* "The actor is the fundamental unit of computation embodying processing, storage, and communication." - Carl Hewitt
* Different from object-orientation since OO is based on heirarcheis of objects based on inheritance. In Akka (and the actor model) however the actors are discrete and uncoupled.

Akka Actors
=========

*General concepts*

* Everything is an Actor
* Each actor has an address for its "Mailbox" (for sending messages to it)
* On receiving a message Actors can; create other actors, send messages to other Actors or change the behaviour for handling the next message
* You never get access to the Actor instance itself, you can only pass messages to its Mailbox
* Mailboxes are like queues that are dealt with asynchronously by the Actor
* An Actor processes one message at a time, which gives the illusion of concurrency.
* Message delivery and processing are separate activities and most probably happen in different threads

*Mutability*

* Actors can have mutable state - Akka takes care of memory consistency (just don't share this mutable state outside the Actor!)
* Messages are immutable.
* The only way to determine an Actors state is to send it a message and see what happens

Actor Systems
==========

* Actors exist together as part of an Actor System - this is part of the Actor model.
* The Actor System provides shared facilities such as thread pools, scheduling, config etc.
* Actors can create child Actors to handle tasks. Parents are then responsible for error handling.
* Each Actor has a parent, even top-level Actors (created with `system.actorOf`) who's parents are special things called "guardians".

        import akka.actor.ActorSystem
        val system = ActorSystem()

Actor implementation
==============

        import akka.actor.{ Actor, ActorLogging }

        // The Actor trait defines the Actor interface
        // ActorLogging provides logging
        class MyActor extends Actor with ActorLogging {

          // The receive method must be overridden and is used
          // to define what messages the Actor can accept.
          // Its generally not a good idea to use wildcards here as
          // it hides cases where unknown messages may be sent to the actor.
          override def receive: Receive = {
            case _ => log.info("Coffee Brewing")
          }
        }

* Sending an unknown message to an Actor is not a fatal error
* Messages are not handled by `receive` but by its return value - there's a lot of magic here.
* Unhandled messages are published to an event stream in the ActorSystem - this can be customized by overriding the `unhandled message`.
* The `receive` method can be set to `Actor.emptyBehavior` to define an Actor that accepts no messages - this could be useful for TDD etc.

*Creating an Actor*

        def createMyActor: ActorRef = {
          // top-level Actor using actorOf
          val myActor = system.actorOf(MyActor.props, "my-actor")
        }

* ***Best practice:*** use a factory method for creating Actors
* The second argument is the actor name and is optional (but preferred). If this is omitted the system will generate a name which will begin with "$"
* Props is the config for the Actor, notably its class.
* ***Best practice:*** define Props in a companion object for the Actor:

        object MyActor {
          def props: Props =
            Props(new MyActor)
          }
        }

* Creating an Actor this way returns an `ActorRef` synchronously. The creation of the Actor itself however is asynchronous! The ActorRef can be used immediately however and the System will queue up messages to be handled by the Actor once it is ready.

*Anonymous Actors*

        system.actorOf(Props(new Actor() with ActorLogging {

          override def receive: Receive = {
            case message => log.info(s"$message")
          }
        }))

* Its possible to create anonymous Actors on the fly which may be useful for testing etc.

Overriding behaviour
==================

override def receive: Receive =
  ready

def ready: Receive = {
  case DoSomething(coffee, guest) =>
    context.become(busy)

    context.system.scheduler.scheduleOnce(
      timeoutDuration,
      self,
      Done
    )
}

def busy: Receive = {
  case Done =>
    anotherActor ! Done
    unstashAll()
    context.become(ready)
  case _ =>
    stash()
}

* The message handler can be swapped on the fly using `context.become()`.
* This can be useful for implementing finite state machines as above.
* Another useful tool here is `stash()` and `unstashAll()`, where when the Actor is set to `busy` then any message other than `Done` will be queued using `stash` (not the same queue as the message inbox!). Once the `Done` message is received after a timeout then we unqueue all messages using `unstashAll` and they are processed as normal after we switch the context back to `ready`.
* Akka actually has a DSL for creating FSMs so it me be worth [reading the docs](http://doc.akka.io/docs/akka/current/scala/fsm.html)

Communication
============

        import akka.actor.{ Actor, ActorLogging }

        class MyActor extends Actor with ActorLogging {

          override def receive: Receive = {
            case Blah => log.info("Blah")
            case Hey(foo) => log.info(s"Hey $foo")
          }
        }

        object MyActor {
          case object Blah
          case class Hey(foo: String)
        }

        // ! pronounced "tell"
        myActor ! Blah

* ***Best practice:*** define a message protocol in the Actor companion object - Don't just pass strings around. When using the message protocol outside of the Actor use the full notation to give a clear namespace e.g. `myActor ! MyActor.Blah`
* Communication is fire and forget!
* The `!` command tries to implicitly include the sender Actor. Using `!` from within another Actor will have the sender in scope already. The sender will default to `Actor.noSender` when none is in scope.
* Messages can be sent back to the sender by using `sender() ! "Some message"`

*Forwarding messages*

* Messages can be forwarded to another Actor using `myActor forward anotherActor`
* Doing so maintains the initial sender, rather than using the forwarding Actor
* Should perhaps be used sparingly as can be a source of confusion
* A more explicit alternative for achieving the same thing is to use the full `tell` method, rather than `!` which takes the sender as a second argument with the message.

Actor Context
============

* Each Actor has an implicit ActorContext
* This can be used to access parent and children, to create child actors, to stop actors etc.

Child actors
===========

        class myActor extends Actor {

          val childActor = createChild()

          def createChild(): ActorRef =
            context.actorOf(Child.props)
        }

* Exactly the same as creating top-level Actors except we use `context.actorOf`, instead of `system.actorOf`. So the child Actor comes from the ActorContext of the parent, rather than from the ActorSystem.

Actor state
==========

* Actors can have mutable and immutable state.
* Immutable state can be set up using constructor arguments:

        class Counter(offset: Int) extends Actor ...

        system.actorOf(Props(new Counter(0)))

* Mutable Actor state can be managed simply using vars. This may seem dangerous since consecutive messages to an Actor may be handled on different threads, but the underlying Akka implementation ensures things remain consistent

* Actors will usually need to send messages to each other and this can be achieved simply by having an Actor take another Actor as an argument:

        // Note that the type of the argument is ActorRef, NOT the class name of the Actor
        class myActor(counter: ActorRef) extends Actor...

Akka Scheduler Service
=================

        // Or global ExecutionContext...
        import context.dispatcher
        import scala.concurrent.duration._

        context.system.scheduler.scheduleOnce(
          2 seconds,
          self,
          SomeMessage
        )

* Can schedule the sending of a message to a particular Actor (in this case the current Actor with `self`).
* More generic version available where execution of an arbitrary block of code can be scheduled.

Testing Actors
==============

*White Box*

        val counter = TestActorRef(new Counter)
        val counterActor = counter.underlyingActor
        counter ! Tick
        counterActor.count shouldEqual 1

* `TestActorRef` can be used to retrieve a version of an actor who's fields are directly accessible (e.g. `count` here, this is not normally possible).
* Synchronous testing.

*Black Box*

        val pingActor = TestProbe()
        val pongActor = system.actorOf(Props(new PongActor(pingActor.ref)))
        pongActor ! PingMessage
        pingActor.expectMsg(PongMessage)

* `TestProbe` allows mock-like Actor instances to be created. We can then assert that certain kinds of messages have been passed to them
* This will be asynchronous so care must be taken. `expectMsg` has a default timeout value.
* If an Actor expects another Actor as an argument, but that second Actor is not important for the current test, `system.deadLetters` can be passed as a sort of "empty Actor". It effectively acts as a DLQ so messages can be sent to it like a proper Actor but they will not be processed.

Starting/Stopping Actors
=============

*Start*

* Creating an Actor automatically starts it

*Stop*

* An Actor can stop itself or be stopped by another Actor

        // stop is defined on the ActorSystem and the current ActorContext
        context.stop(self)
        context.stop(other)

* A stopped Actor will no longer process messages
* An Actor will finish any in flight messages on receiving a Stop command; it will then stop message processing; it will then recursively stop its children; it can then terminate itself which frees it up for Garbage Collection
* If an Actor needs to be stopped more gracefully than this, it should define a message handler specifically for that. It can then do any tasks required before being stopped and then stop itself.
* An Actor can observe another Actor and react to its termination using `context.watch(otherActorRef)`. The observing Actor will receive a `Terminated(otherActorRef)` message if the observed Actor terminates.

*Hooks*

* Actors have `preStart` and `postStop` hooks that can be used to define bahaviour at these points in the Actor's lifetime (immediately after starting before processing any messages, immediately before termination).

Failure
======

* Akka's philosophy is let it crash!
* An Actor fails when it throws an Exception, which can happen in three places:
** during message processing
** during initialization
** within a lifecycle hook, e.g. preStart
* By default, if an Exception is thrown, the Actor will be restarted so as not to bring down the whole system. This is called ***parental supervision***

*Parental Supervision*

* When a child throws an Exception its message processing is suspended, its children and their descendants are suspended and the parent has to handle the failure.
* Parent failure handling is defined by its `supervisorStrategy`. The default supervisorStrategy is to restart the child. In most cases this behaviour should be overridden.

*Supervisor Strategies*

* Two high level strategies are `OneForOneStrategy` - only the failing child is affected, `AllForOneStrategy` - all children are affected when one fails.
* These can be configured more specifically with a `Decider` which maps specific failures to a particular Directive.
* If the supervisor does not define how to handle a particular failure, the supervisor itself is considered to be faulty.

* If you don't override supervisorStrategy, a `OneForOneStrategy` with the following decider is used by default:
** ActorInitializationException - Stop
** ActorKilledException - Stop
** DeathPactExceptions - Stop
** Other Exceptions - Restart
** Other Throwables - Escalates to it's parent

*Directives*

Available directives are:

* Resume
** continue processing messages with same instance.
** No messages are lost except the faulty one.
** Actor state is not changed so use Resume if state is considered valid.
* Restart
** replace Actor with a new instance.
** No messages are lost except the faulty one.
** Actor state is lost so use Restart if state is considered invalid
** By default all children get stopped. This is because if a parent creates its child as part of the constructor, to simply restart the child would result in a duplicate child when the failing parent restarts and creates its child!
* Stop - stop the Actor
* Escalate - Delegate the decision to the supervisor's parent

Remember that all Actor communication is performed via ActorRefs. So if we restart an Actor, the ActorRef remains in place but the Actor behind it is replaced with a new instance. This means that all existing references to the Actor are still valid after restarting.

*Custom Supervisor Strategies*

        override val supervisorStrategy: SupervisorStrategy =
          OneForOneStrategy() {
            val decider: Decider = {
              case MyActor.SomeException => SupervisorStrategy.Stop
            }

            decider orElse super.supervisorStrategy.decider
          }

* In a parent Actor override `supervisorStrategy`.
* Chose either a `OneForOneStrategy` or `AllForOneStrategy`.
* The first argument list can be used to override the maximum number of retries and the time range these are allowed to occur in. They have defaults though so the arguments can be left out.
* The second argument is a `Decider` which is a `PartialFunction[Throwable, Directive]` where the Directives mentioned above can be found in the SupervisorStrategy object.
* In order to define a default handled for all other Throwables the `super.supervisorStrategy.decider` can be used as above. This is the default Decider that we are now overriding.
* Within the `Decider` you also have access to `sender()` being the ActorRef of the Actor that failed. You can also therefore `forward` messages.

Self Healing
===========

* Akka is commonly used to create self healing Systems
* This can be achieved if the supervisor dealing with a child's failure has all the necessary information to recreate the state and/or messages necessary to for the application to continue.
* A simple approach is to define custom Exceptions for Actors to throw, which can contain the important information the Actor has access to when it fails. The supervisor can then use this information to send appropriate messages before restarting/stopping/whatever the failing child.


        override val supervisorStrategy: SupervisorStrategy =
          OneForOneStrategy() {
            val decider: Decider = {
              case MyActor.SomeException(state1, state2) =>
              anotherActor.tell(AnotherActor.SomeMessage(state1, state2), sender())
              SupervisorStrategy.Restart
            }

            decider orElse super.supervisorStrategy.decider
          }

* Above the supervisor receives a `MyActor.SomeException` from a child. The Exception contains to bits of information, `state1` and `state2`. The supervisor uses this information to send a message to another Actor, allowing the system to continue in some manor (this is probably what the child should have done if it hadn't failed). The supervisor finishes by restarting the faulty child and the system can then continue as normal.
* note the use of `tell` here. So for `AnotherActor` the sender of the message will be the Actor that failed (since in the supervisorStrategy the Actor that failed is also the sender)

Routers and Dispatchers
======================

* Actors process one message at a time
* To work in parallel you have to use multiple actors in parallel

*Concurrency vs Parallelism*

* Tasks are concurrent when their execution order is not defined (non-deterministic) - they are not necessarily executed in parallel
* Concurrent programming is primarily concerned with the complexity that arises due to non-deterministic control flow
* Parallel programming aims at improving throughput and making control flow deterministic

*Routers*

Routers allow an Akka system to execute messages in parallel

* Routers are special Actors that deliver messages to a number of routee Actors.
* There are two types of Routers; Pool routers, which create routees as child Actors, and Group routers, which require the Actor paths to be passed to them (routees must therefore be created in advanced)
* Pool routers are able to scale up and down based a `Resizer`.
* A number of routing strategies are available including `RandomRoutingLogic` and `RoundRobinRoutingLogic`
* All messages are delivered to one routee only except the following:
** `PoisonPill` is not delivered to any routee
** `Kill` is not delivered to any routee
** The payload of `Broadcast` is delivered to all routees

        context.actorOf(
          Props(new SomeActor).withRouter(FromConfig()),
          "some-actor"
        )

        // Example config file
        actor {
          deployment {
            /coffee-house/barista {
              router = round-robin-pool
              nr-of-instances = 3
            }
          }
        }

        context.actorOf(
          RoundRobinPool(nrOfInstances = 5).props(Props(new SomeActor)),
          "some-actor"
        )

* Routers can be implemented either using `Props.withRouter` and using a config file *or* by using a particular router constructor e.g. `RoundRobinPool(nrOfInstances = 5)`

*Dispatchers*

Dispatchers allow the parallelism of a system to be optimised. They do this by configuring the ExecutionContext:

        akka {
          actor {
            default-dispatcher {
              fork-join-executor {
                parallelism-min = 4
                parallelism-factor = 2.0
                parallelism-max = 64
              }
              throughput = 5 // default
            }
            ...

* The number of threads available for parallel tasks is determined by `number of cores * parallelism-factor` clipped by `parallelism-min` and `parallelism-max`.
* The throughput defines how many messages may be routed to a routee. If all routee mailboxes have the maximum number of messages then the message must be queued at the router. Higher throughput values may increase the global throughput of the system but increase latency at individual routees. Lower values may reduce latency but increase throughput.


Ask Pattern
==========

* The Ask pattern is used to interact with Actors from outside of the Actor system.
* The response is returned as a Future

        class PongActor extends Actor with ActorLogging {
          override def receive: Receive = {
            case PingMessage => sender() ! PongMessage
          }
        }

        val response = pongActor ? PingMessage

        response.mapTo[PongMessage] onComplete {
          case Success(...) => ...
          case Failure(...) => ...
        }

* Due to type erasure (don't ask), the response is a `Future[Any]`. This should be mapped to something more concrete using `mapTo`. The future can then be dealt with as usual.
* The Actor being asked the question simply needs to send a message back to the sender.
