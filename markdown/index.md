As I write this, I have a basic understanding of _functions as a service_ and _serverless architectures_, but as of about a month
ago, I could not comprehend what purpose these things serve or why I'd ever care about them.  It's such a different way of
thinking about building applications that it just didn't compute.

But, I think there's something to it, and I think having an understanding of _what problem it solves_ will make it clear that an architecture built on functions managed by “someone else” could lead to powerful system design.  Let's learn it together.

Functions as a services or _FaaS_ is the generic name for products like [AWS Lambda](https://aws.amazon.com/lambda/), which bills
itself as “Serverless Compute”, which is a fancy way of saying “you can run your code without having to manage even _virtual_
servers”, followed by a *big huge* asterisk of restrictions and constraints around how you write your code and what it can do.

Some of those are outlined on Lambda's [limits page](http://docs.aws.amazon.com/lambda/latest/dg/limits.html), such as:

* You don't get more than 1.5G of memory
* Your code can't run for more than 300 seconds
* Your app can't be bigger than 50MB

The idea is that your code is a small function that does one thing, does it quickly, and doesn't require a lot of resources.  If
you can arrange for that to happen, you can run your code without managing a server of any kind.  This is why FaaS is used
interchangeably with the word _serverless_ (not to be confused with the confusingly-named-but-seriously-points-for-calling-dibs-on-the-name [Serverless Framework](https://serverless.com), which we will only refer to by its full and capitalized name).

You might wonder how your function even gets called.  This, too, has lots of limitations and, at first, feels really strange.
When you look at AWS' [use cases](http://docs.aws.amazon.com/lambda/latest/dg/use-cases.html), it talks about events, and FaaS or
serverless architecture are *also* talked-about with phrases like “event sourcing”.

And *this* is the key insight into what all this is about.  If you choose to think of your system as one that receives
events—like a user clicking on a web page, or a timer going off, or a database being updated, and *your* code is nothing but a
bunch of functions that are invoked based on these events, it starts to almost make sense.  Almost.

What we're going to try to do is look at some common use cases where you'd set up some application to run on some server such that
it does some things, and re-imagine those use-cases as _event sourced_ and implemented as functions.  We'll see where that takes
us.
