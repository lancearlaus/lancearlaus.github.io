---
published: true
layout: post
summary: Save yourself a deadlock headache by knowing when to use buffers in your flows
categories: scala akka streams reactive
title: "Akka Streams Beyond Basics: Keep that Buffer handy to avoid deadlock"
---

Deadlock in an Akka Streams application? Yes, and it doesn't take much, but it's also easy to avoid for the case covered here.

In this post, I'll give you a quick tip for avoiding deadlock in your branching flows and share a simple, real-world example that demonstrates its use.

First, the quick tip...

#### Quick Tip: Use a Buffer to match uneven flow branches

__Summary:__  
Use a buffer if you broadcast a stream and subsequently zip the resulting outputs together if the intermediate branches are uneven.

<aside>Hint: If your flow looks like a diamond, you'll need a buffer if the branches are uneven.</aside>

__Explanation:__  
A standard Broadcast stage branches a stream. It has one input and multiple outputs with each output emitting the original input. In other words, a standard Broadcast stage broadcasts each incoming element to multiple recipients.

Uneven flow branches are created when elements are stored, created, or dropped differently amongst branches. A simple example, depicted below, is a two-branch stream where elements are dropped on one branch, but not the other.

 

#### A Good Starting Point

Unix pipes are a convenient analogy to use when explaining Reactive Streams to developers. After all, who hasn't strung together a couple of commands with the trusty pipe (|) operator on the command line?

````bash
# Count my Scala files
find ~/Projects -name '*.scala' | wc -l
````

However, the analogy only goes so far and, in fact, can get in the way when trying to understand a Reactive Streams-based application. At least it did for me, hence the post.

#### The Key Difference - Pull vs. Push, or Why a Stream is NOT a Pipe

I know what you're thinking 

#### A Simple Problem

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
