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

<div class="mxgraph" style="position:relative;overflow:auto;width:100%;"><div style="width:1px;height:1px;overflow:hidden;">7Vpdb+I6E/41SOdcFCUkfF2Wds+H9FZvpV7stYkNWGtijmMo3V+/M8k4iZPAtqv0CB3gApLH8djzzEfsMYPoYXv807Dd5klzoQajgB8H0eNgNJpPQ/hG4K0AZrNxAayN5AVETyDwIr8LAgNC95KLzHvQaq2s3PlgotNUJNbDVlr5Q+zY2omvgJeEqTb6VXK7oSmPJhX+l5DrjRsmnMyLlsy+ORlcrNhe2bscgjZs3jIni7Q6BsVtTP3f6D6MZm6o1JvSd623HmBEVnFF6kpf/ZQdvEGX2nBhPEjJ9Fudx+gLmNJoDYLwant8EArN6UxVdPvjRGtJnxEpTeV8hzgZhywSnCWxiEQwuSOiD0ztSbfBaKJA1oLLg8fz5J89znJhxdHeMSXXwNc9aiRWtmqFqzX95lIy5LVLDBAh7jZkW5QTDnHohpgXJblM19D8VaZcv8JFLiWI3ACgazGGPy7AuQIObfiM0fuUCyQlhObXjbTiZccSbH2FqAJsY7eKmlc6tRQnECTRgpmEbudwdxDGSvDo/7GlUM86k1ZqdCWrUYxqoAkYCl1iQQw+5vRVUu4JXmpr0QEXnGWbcqJ488wsSMBnwPpBXGpXdwLyCxQqKC9U3gaJQ+itsOYN48KFASUJShrhnPz1tYrK0ZQwZzUEJ4Qxcup1KbryQrggR+x2SorZDh+0bAkWa5jWLjXktiZoWgh/l/uaXJeW4z0yC/3ggdKdUN4vjdAdIJhugyeGipweAcG2Yj2r+mwkOv4n6uqELCvlhwHoX45XNnycDQS7HAJx33v6TAE9hFzkXgkUciNkpBFys46Ii3uIuNZrANLp9YUgLAWuNgLntwAcRRcUgPE1BiAsvW+vwCuOwNCF0iVEIC2ArysCp1ccgbNbAI4vKP6oDuPFX4OubMN2eLlS4nhvDJQDooVIOV0+JoplmUzOs+ZUi4e4dYbe1DgdxlE4r33GRbMrS82H0eQc6YJ7la425TVOHe11Th1mhGJWHvxCUxfRNMKzljCTalEz9Tfy5abdicj03iSCetUrRQ1BcfATQZaZtbAtQbndS7V/zRVcKfJ0Lg5c+aRWUDob9beiwW3F9DM2qOT4dP9b9Pu/vRubQCH2ul9FjbVgVwXSQX0XINsJyE3/loBuJZP+SyaXl36ufiUcX1T66aoGNQjD87P8lM7AIWxxtgRLO5zq2aMrylrlMVQf1DUK2eWuvkadW9HVqQvdkrdf7tr7+L/T3b5Yo9YJxFjxycqs0d/Eg1Yaz2xTnRO8kko1IEdikZraZ3dbyTmO0unGGp5eqXy/soHnBHTwrIS7kvyeZtpxxPvxo4ZJ43Svw0IOqlvIGbZfA9EJfM1A/9/bz7HQicPVizNQK/t0GMhlmk83kBP6n92JnyC93OG+c6f+/k35yZE+uikfT3EPVH1I49636HBb/SeleLz6r1H05Qc=</div></div>

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
