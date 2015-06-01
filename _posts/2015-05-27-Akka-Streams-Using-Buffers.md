---
published: true
layout: post
summary: Save yourself a deadlock headache by knowing when to use buffers in your flows
categories: scala akka streams reactive
title: "Akka Streams Beyond Basics: Keep that Buffer handy to avoid deadlock"
---


![Akka Streams Buffers Thumbnail]({{ site.url }}/images/Akka Streams Buffers Thumbnail.png)

Deadlock in an Akka Streams application? Yes, and it doesn't take much, but it's also easy to avoid for the case covered here.

In this post, I'll give you a quick tip for avoiding deadlock in your branching flows and share a simple, real-world example that demonstrates its use.

#### Tip: Use a Buffer to match uneven flow branches

Branches are created when splitting a stream. The `Broadcast` stage, for example, emits each incoming element to multiple recipients, creating a branch for each of its outputs. A common Reactive Streams pattern is to process these branches and combine the results to yield a new, enhanced output.

__TODO: Diagram__

So far, so good. However, the potential for deadlock arises when branches emit elements at different rates. That is, if there exists any branch that doesn't emit an element every time another branch emits an element. These mismatched, or uneven, branches will stall a downstream stage, like Zip, that waits for all inputs to arrive before emitting.

Branches can become uneven when one branch stores, drops, or creates elements. This is a relatively common case, and you'll need a buffer on the longer branch(es) to absorb the slack and balance the flow.

It's a situation that's perhaps best explained by example.

#### A Simple Example

Here's a simple problem to solve using Akka Streams.

> Given a reverse time series of daily integers (that is, most recent number first), calculate the 7-day trailing difference

The solution is a branched flow containing a drop to create the offset.

````scala
def trailingDifference(offset: Int) = Flow() { implicit b =>

  val bcast = b.add(Broadcast[Int](2))
  val drop = b.add(Flow[Int].drop(offset))
  val zip = b.add(Zip[Int, Int])
  val diff = b.add(Flow[(Int, Int)].map { case (num, trailing) => (num, num - trailing) })

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

The resulting stream of tuples contains the original number and the trailing difference. Astute readers will note that the output stream will be shorter than the input stream since the difference can't be calculated for the trailing elements.

#### Why Deadlock?

The code above looks fine and may run just fine for small differences (hey, my tests pass!). However, it's got a subtle deadlock bug that will rear its ugly head intermittently, the worst sort of bug to wrangle in production.

Deadlock happens due to the behavior of `Broadcast` and `Zip`. `Broadcast` won't emit an element until all outputs (branches) signal demand while `Zip` won't signal demand until all its inputs (branches) have emitted an element. Uneven branches create a flow where the `Broadcast` stage waits for demand that's never signalled while the `Zip` stage waits for an element that never arrives.

Let's apply the basic rule of Reactive Streams to understand why.

> Rule: Producers emit elements in response to demand

Sounds familar, right? This basic rule is, of course, the essence of push-based systems that use demand-based back pressure.

Here's what happens in our simple example.

1. The `Sink` signals demand, which is relayed through the flow to the `Source`
2. The `Source` receives the demand signal and emits an element that travels through the `Broadcast` stage and down both branches
3. The `Zip` stage receives an element from the branch without the `Drop`. However, it does __NOT__ receive an element from the branch with the drop. `Zip` must wait until it receives all inputs to emit an element downstream (it must emit well-formed tuples).
4. The `Drop` stage on the other branch drops the element it received and signals demand upstream to the `Broadcast` stage.
5. The `Broadcast` stage now has one branch (the one with the `Drop`) that has signalled demand, and one branch that has not since it's connected to the `Zip` stage that's waiting for an element from the `Drop` stage.

Voila, deadlock!

<iframe src="http://clips.animatron.com/f0fa6620afaf9dbb11b9890562b7bb26?w=758&h=240&t=0&r=1" width="758" height="240" frameborder="0"></iframe>

Use a buffer if you broadcast a stream and subsequently zip the resulting outputs together if the intermediate branches are uneven.

<aside>Hint: If your flow looks like a diamond, you'll need a buffer if the branches are uneven.</aside>

#### Quick Tip Explained

A standard Broadcast stage branches a stream. It has one input and multiple outputs with each output emitting the original input. In other words, a standard Broadcast stage broadcasts each incoming element to multiple recipients.

Uneven flow branches are created when elements are stored, created, or dropped differently amongst branches. A simple example, depicted below, is a two-branch stream where elements are dropped on one branch, but not the other.

__TODO: Diagram__

Notice the difference in the number of elements produced 

#### A Real-World Example

Here's a simple, real world problem to solve using Akka Streams.

> Given a reverse time series of stock quotes (that is, most recent quote first) and a sliding window size, compute the simple moving average (SMA) of the closing price for each quote in the series

![Akka Streams Moving Average Data]({{ site.url }}/images/Akka Streams Moving Average Data.png)

Curious where this problem comes from? Historical stock quotes are often delivered as a reverse time series (Yahoo Finance does this, for example) since recent data is typically most relevant

#### A Simple Solution

Here's a simple solution to the problem.

![Akka Streams Moving Average Flow]({{ site.url }}/images/Akka Streams Moving Average Flow.png)

* Copy (broadcast) the incoming stream into two streams. Let's call them Up and Down.
* Leave the Up stream unchanged
* Queue a sliding window of prices and calulate the moving average on the Down stream
* Zip and merge the Up and Down stream elements together

The result is a stream of quotes with the simple moving average appended. Here's the corresponding Scala code. I covered the sliding window stage in a previous post on creating stateful stages. Note that this example was coded against Akka Streams 1.0-RC3 and that it uses the flow DSL (~>) operator.

````scala
def movingAverage(window: Int = 3) = Flow() { implicit b =>

  val in = b.add(Broadcast[Quote](2))
  val extract = b.add(Flow[Quote].map(_("Close").toDouble))
  val sliding = b.add(Flow[Double].transform(() => new Sliding(window)))
  val average = b.add(Flow[Iterator[Double]].map(_.foldLeft(0.0)(_ + _)).map(_ / window))
  val zip = b.add(Zip[Quote, Double])
  val merge = b.add(Flow[(Quote, Double)].map { case (quote, sma) => quote + ("SMA" -> f"$sma%1.2f") })

  in ~>                                  zip.in0
  in ~> extract ~> sliding ~> average ~> zip.in1
                                         zip.out ~> merge

  (in.in, merge.outlet)
}
````


#### Invert that Thought

I know what you're thinking - you've already read the basics on streams. You get that it's 

To really understand Reactive Streams, you need to stop thinking

If you're reading this, you're probably familiar with Akka Streams and the Reactive Streams movement. You've probably also 

There is a significant amount of subtle, yet precisely calibrated, styling to ensure
that your content is emphasized while still looking aesthetically pleasing.

All links are easy to [locate and discern](#), yet don't detract from the harmony
of a paragraph. The _same_ goes for italics and __bold__ elements. Even the the strikeout
works if <del>for some reason you need to update your post</del>. For consistency's sake,
<ins>The same goes for insertions</ins>, of course.

### Code, with syntax highlighting

Code blocks use the [solarized](http://ethanschoonover.com/solarized) theme. Both the light and
dark versions are included, so you can swap them out easily. _Solarized Dark_ is the default.

````scala
  def bollinger(window: Int = defaultWindow) = Flow() { implicit b =>

    val in = b.add(Broadcast[Quote](2))
    val extract = b.add(Flow[Quote].map(_("Close").toDouble))
    val statistics = b.add(Flow[Double].slidingStatistics(window))
    val bollinger = b.add(Flow[Statistics[Double]].map(Bollinger(_)))
    // Need buffer to avoid deadlock. See: https://github.com/akka/akka/issues/17435
    val buffer = b.add(Flow[Quote].buffer(window, OverflowStrategy.backpressure))
    val zip = b.add(Zip[Quote, Bollinger])
    val merge = b.add(Flow[(Quote, Bollinger)].map(t => t._1 ++ t._2))

    in ~> buffer                             ~> zip.in0
    in ~> extract ~> statistics ~> bollinger ~> zip.in1
                                                zip.out ~> merge

    (in.in, merge.outlet)
  }
````

# Headings!

They're responsive, and well-proportioned (in `padding`, `line-height`, `margin`, and `font-size`).
They also heavily rely on the awesome utility, [BASSCSS](http://www.basscss.com/).

##### They draw the perfect amount of attention

This allows your content to have the proper informational and contextual hierarchy. Yay.

### There are lists, too

  * Apples
  * Oranges
  * Potatoes
  * Milk

  1. Mow the lawn
  2. Feed the dog
  3. Dance

### Images look great, too

![desk](https://cloud.githubusercontent.com/assets/1424573/3378137/abac6d7c-fbe6-11e3-8e09-55745b6a8176.png)


### There are also pretty colors

Also the result of [BASSCSS](http://www.basscss.com/), you can <span class="bg-dark-gray white">highlight</span> certain components
of a <span class="red">post</span> <span class="mid-gray">with</span> <span class="green">CSS</span> <span class="orange">classes</span>.

I don't recommend using blue, though. It looks like a <span class="blue">link</span>.

### Stylish blockquotes included

You can use the markdown quote syntax, `>` for simple quotes.

> Lorem ipsum dolor sit amet, consectetur adipiscing elit. Suspendisse quis porta mauris.

However, you need to inject html if you'd like a citation footer. I will be working on a way to
hopefully sidestep this inconvenience.

<blockquote>
  <p>
    Perfection is achieved, not when there is nothing more to add, but when there is nothing left to take away.
  </p>
  <footer><cite title="Antoine de Saint-Exupéry">Antoine de Saint-Exupéry</cite></footer>
</blockquote>

### There's more being added all the time

Checkout the [Github repository](https://github.com/johnotander/pixyll) to request,
or add, features.

Happy writing.
