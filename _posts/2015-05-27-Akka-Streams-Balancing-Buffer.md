---
published: true
layout: post
title: "Akka Streams: Using a Balancing Buffer to Avoid Deadlock"
summary: "Save yourself a deadlock headache by knowing when to use the Balancing Buffer pattern in your flows"
categories: akka streams scala
---


![Akka Streams Buffers Thumbnail]({{ site.url }}/images/Akka Streams Buffers Thumbnail.png)

Deadlock in an Akka Streams application? Yes, and it doesn't take much, but it's also easy to avoid for the case covered here.

In this post, I'll cover the Balancing Buffer pattern for avoiding deadlock in your branching flows and share a simple, real-world example that demonstrates its use. This post assumes basic familiarity with Akka Streams and Scala.

## Background: Unbalanced Flow Branches

Branches are created when splitting a stream. The `Broadcast` stage, for example, emits each incoming element to multiple recipients, creating a branch for each of its outputs. A common pattern found in Reactive Streams applications is to process these branches and combine the results into a new output.

![Akka Streams Diamond Flow]({{ site.url }}/images/Akka Streams Diamond Flow.png)

So far, so good. However, the potential for deadlock arises when branches emit elements at different rates. That is, if there exists any branch that doesn't emit an element every time another branch emits an element. These uneven, or unbalanced, branches will stall a downstream stage, like Zip, that waits for all inputs to arrive before emitting.

Branches can become unbalanced when one branch drops or stores elements. For example, a sliding window stage that buffers elements before emitting a full window. This is a relatively common case, and you'll need a Balancing Buffer on the corresponding branch(es) to absorb the slack and balance the flow.

## Balancing Buffer Design Pattern Explained

A Balancing Buffer is a buffer stage inserted into a flow to balance, or match, its demand with another flow, thus avoiding deadlock by ensuring that sufficient demand is signalled upstream to continue the flow of elements through the system. The Balancing Buffer is inserted immediately after the `Broadcast`, or other fan-out, stage and is sized to the maximum offset between branches.

Follow the guidelines below to avoid deadlock by using a Balancing Buffer in your branching flows.

1. Identify flows that fan out and fan in multiple branches, typically via a `Broadcast` and subsequent `Zip` stage.
2. Identify unbalanced branches and determine the maximum offset between them. The maximum offset is the maximum number of elements emitted by one branch before a corresponding branch emits an element.
3. Insert a buffer immediately after the `Broadcast` stage on the branch with slack and size it to the maximum offset.

## A Simple Example

Let's look at a simple example to understand why deadlock occurs and how to avoid it using a Balancing Buffer.

> Given a reverse time series of daily integers (that is, most recent number first), calculate the 7-day trailing difference

The solution is a branched flow containing a drop to create the offset.

````scala
def trailingDifference(offset: Int) = Flow() { implicit b =>

  val bcast = b.add(Broadcast[Int](2))
  val drop = b.add(Flow[Int].drop(offset))
  val zip = b.add(Zip[Int, Int])
  val diff = b.add(Flow[(Int, Int)].map { 
    case (num, trailing) => (num, num - trailing) 
  })

  bcast ~>         zip.in0
  bcast ~> drop ~> zip.in1
                   zip.out ~> diff

  (bcast.in, diff.outlet)
}

// Example usage for 7-day trailing difference that prints:
// (100, 14)
// (98, 14)
// ...
Source(100 to 0 by -2).via(trailingDifference(7)).runWith(Sink.foreach(println))

````

The resulting stream of tuples contains the original number and the trailing difference. Note that the output stream will be shorter than the input stream since the difference can't be calculated for the trailing elements.

## Why Deadlock?

The code above looks fine and may run just fine for small differences (hey, my tests pass!). However, it's got a subtle deadlock bug that will rear its ugly head intermittently, the worst type to wrangle in production.

Deadlock occurs due to the `Drop` combined with the behavior of `Broadcast` and `Zip`. `Broadcast` won't emit an element until all outputs (branches) signal demand while `Zip` won't signal demand until all its inputs (branches) have emitted an element. This makes sense since `Broadcast` is constrained by the slowest consumer and `Zip` must emit well-formed tuples.

Unbalanced branches create a flow where the `Broadcast` stage waits for demand that's never signalled while the `Zip` stage waits for an element that never arrives.

> Rule: Producers emit elements in response to demand

Sounds familar, right? This basic rule of Reactive Streams is the essence of push-based systems that use demand-based back pressure. Applying it to our example, here's a detailed look at what happens.

<iframe src="http://clips.animatron.com/693b6b4c865ed7d0344d6ab4276b7d59?w=758&h=240&t=0&r=1" width="758" height="240" frameborder="0"></iframe>

1. The `Sink` signals demand, which is relayed through the flow to the `Source`
2. The `Source` receives the demand signal (D) and emits an element that travels through the `Broadcast` stage and down both branches
3. The `Zip` stage receives an element from the branch without the `Drop`. However, it does __not__ receive an element from the `Drop` branch. `Zip` must wait until it receives all inputs to emit an element downstream (remember, it must emit well-formed tuples).
4. The `Drop` stage on the other branch discards the element and signals demand upstream to the `Broadcast` stage.
5. The `Broadcast` stage now has one branch (the one with the `Drop`) that has signalled demand, and one branch that has not since it's connected to the `Zip` stage that's waiting for an element from the `Drop` stage.

Voila, deadlock!

## Avoiding Deadlock

We can apply the guidelines covered earlier to balance our flow and avoid deadlock. The maximum difference between the two branches in our case is the trailing difference offset. The buffer is sized accordingly.

````scala
def trailingDifference(offset: Int) = Flow() { implicit b =>

  val bcast = b.add(Broadcast[Int](2))
  // Add a balancing buffer
  val buffer = b.add(Flow[Int].buffer(offset, OverflowStrategy.backpressure))
  val drop = b.add(Flow[Int].drop(offset))
  val zip = b.add(Zip[Int, Int])
  val diff = b.add(Flow[(Int, Int)].map { 
    case (num, trailing) => (num, num - trailing) 
  })

  bcast ~> buffer ~> zip.in0
  bcast ~> drop   ~> zip.in1
                     zip.out ~> diff

  (bcast.in, diff.outlet)
}
````

That's it - a two line change to avoid deadlock.

## Addendum: Why Intermittent Deadlock?

You may have noticed that I used the word 'intermittent' when describing the deadlock bug above.
Your tests may run fine 9 out of 10 times, only to fail that 10th time without you being able to reproduce the problem in a debugger.
Deadlock is bad enough, but intermittent bugs are truly nasty, the worst kind, and should be avoided by design.

What makes this bug intermittent?

Hint: The answer lies in the internal buffers that Akka Streams sets up by default to improve performance, but that sounds like fodder for another blog post...

## Parting Words and Acknowledgements

I hope this article helps you avoid the deadlock headache I experienced. Heck, even the Akka folks from Typesafe weren't entirely familiar with this issue when I [posted to their mailing list](https://groups.google.com/forum/#!topic/akka-user/SE8l9oxGjtY).

Thanks to Endre Varga ([@drewhk](https://twitter.com/drewhk)) for his response to [the issue I raised](https://github.com/akka/akka/issues/17435) that helped me understand the issue at hand.
