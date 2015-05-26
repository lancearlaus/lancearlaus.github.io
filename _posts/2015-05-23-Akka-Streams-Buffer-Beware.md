---
published: true
layout: post
summary: "Want to really understand Reactive Streams? Stop thinking Unix pipes."
categories: scala akka streams reactive
title: "Beyond the Pipe: Really Thinking Pull in Reactive Systems"
---



Want to really understand Reactive Streams? Stop thinking Unix pipes.

In this post, I'll tell you why the Unix pipes analogy can get in the way of understanding and debugging an Akka Streams (or any Reactive Streams) application.

#### A Good Starting Point

Unix pipes are a convenient analogy to use when explaining Reactive Streams to developers. After all, who hasn't strung together a couple of commands with the trusty pipe (|) operator on the command line?

````bash
# Count my Scala files
find ~/Projects -name '*.scala' | wc -l
````

However, the analogy only goes so far and, in fact, can get in the way when trying to understand a Reactive Streams-based application. At least, it did for me, hence the post.

#### A Simple Problem

Here's a simple, real world problem to solve using Akka Streams.

> Given a reverse time series of stock quotes (that is, most recent quote first) and sliding window size, compute the simple moving average for each point in the series

Curious where this problem comes from? Historical stock quotes are often delivered as a reverse time series (Yahoo Finance does this, for example) since recent data is typically most relevant

#### A Simple Solution

Here's a simple solution to the problem

* Copy (broadcast) the incoming stream into two streams. Let's call them Up and Down.
* Leave the Up stream unchanged
* Queue a sliding window and calulate the moving average on the Down stream
* Zip and merge the Up and Down stream elements together
* The final output is a stream of quotes with the simple moving average appended as an additional field

<div class="mxgraph" style="position:relative;overflow:auto;width:100%;display:inline-block;_display:inline;"><div style="width:1px;height:1px;overflow:hidden;">7Vpdb+I6E/41SOdcFCUkfF2Wds+H9FZvpV7stYkNWGtijmMo3V+/M8k4iZPAtqv0CB3gApLHztjzzEfsMYPoYXv807Dd5klzoQajgB8H0eNgNJpPQ/hG4K0AZrNxAayN5AVEPRB4kd8FgQGhe8lF5nW0Wisrdz6Y6DQVifWwlVb+EDu2duIr4CVhqo1+ldxuaMqjSYX/JeR644YJJ/OiJbNvTgYXK7ZX9i6HoA2bt8zJIq2OQXEb0/NvdB9GMzdU6k3pu9ZbDzAiq7gidaWvfsoO3qBLbbgwXhcl0291HqMvYEqjNQjCq+3xQSg0pzNVIemPE60lfUakNJXzD8TJOGSR4CyJRSSCyR0RfWBqT7oNRhMFshZcHjyeJ//scZYLK472jim5Br7uUSOxslUrXK3pN5eSIa9dYoAIcbch26KccIhDN8S8KMlluobmrzLl+hUucilB5AYAXYsx/HEBzhVwaMNnjN6nXCApITS/bqQVLzuWYOsrRBVgG7tV1LzSqaU4gSCJFswkdDuHu4MwVoJH/48thXrWmbRSoytZjWJUA03AUOgSC2LwMaevknJP8FJbiw644CzblBPFm2dmQQL2AesHcald3QnIL1CooLxQeRskDqG3wpo3jAsXBpQkKGmEc3Lh1yoqR1PCnNUQnBDGyKnXpejKC+GCHLHbKSlmO3zQsiVYrGFau9SQ25qgaSH8Xe5rcl1ajvfILDwHHUp3Qnm/NEJ3gGC6DZ4YKnJ6BATbivWs6rOR6PifqKsTsqyUHwagfzle2fBxNhDscgjEfe/pMwX0EHKRe0tQyI2QkUbIzToiLu4h4lqvAUin1xeCsBS42gic3wJwFF1QAMbXGICw9L69Aq84AkMXSpcQgbQAvq4InF5xBM5uATi+oPijOowXfw26sg3b4eVKieO9MVAOiBYi5XT5mCiWZTI5z5pTLR7i1hmepsbpMI7Cee0zLppdWWo+jCbnSBfcq3S1Ka9x6mivc+owIxSz8uAXmrqIphGetYSZVIuaqb+RLzftTkSm9yYR9FS9UtQQFAc/EWSZWQvbEpTbvVT711zBlSJP5+LAlU9qBaWzUX8rGtxWTD9jg0qOT/e/Rb//27uxCRRir/tV1FgLdlUgHdR3AbKdgNz0bwnoVjLpv2Ryeenn6lfC8UWln65qUIMwPD/LT+kMHMIWZ0uwtMOpnj26oqxVHkP1QV2jkF3u6mvUuRVdnbrQLXn75a69j/873e2LNWqdQIwVn6zMGv1NPGil8cw21TnBK6lUA3IkFqmpfXa3lZzjKJ1urKH3SuX7lQ30E/CAZyXcleT3NNOOI96PHzVMGqd7HRZyUN1CzrD9GohO4GsG+v/efo6FThyuXpyBWtmnw0Au03y6gZzQ/+xO/ATp5Q73nTv192/KT4700U35eIp7oOpDGve+RYfb6j8pRffqv0bRlx8=</div></div>

And, the corresponding Akka Streams code. Note that this example was coded against Akka Streams 1.0-RC2 and that it uses the flow DSL (~>) operator

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

<script type="text/javascript" src="https://www.draw.io/js/embed-static.min.js"></script>
