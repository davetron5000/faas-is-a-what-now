It's not lost on me that we've yet to really deploy a function or serverless application yet.  We're only trying to find out what
problem this sort of architecture solves.  I think it's this:

<aside class="pullquote">You are responsible for your code, and everything else is handled by the provider</aside>
The less code you are responsible for running in production, the better. Running an actual server means it's all on you.  Using virtual servers delegates some to your hosting provider.  Using a platform-as-a-service like Heroku delegates yet more.  A serverless architecture reduces this to what might be the absolute minimum - you are responsible for your code, and everything else is handled by the provider.  This means, if you trust your provider's competence and support, you can spend most of your time and attention on *your* system and the problems it exists to solve.

Event-sourcing is simply a natural consequence of writing code in this way, because functions must respond to inputs, and inputs
are easily modeled as a series of events.

Where do we go from here?  A blog, todo list, or Hacker News clone doesn't feel like a good demonstration of this technology, but
there could be some benefit to a reference system, like Java's venerable [Pet
Store](http://www.oracle.com/technetwork/java/index-136650.html).  The code we wrote here is simple enough to understand the
concepts, but it's still contrived.  [AWS has a reference application](https://github.com/awslabs/lambda-refarch-webapp) and *wow* is it complex.
Just to even understand it, you need to know what Cloudfront, S3, DynamoDB, Cognito, STS, Route 53, and API Gateway are.

This is all begging for a framework, because you'll end up managing more configuration of these managed services than you will
actual code.  And configuration is notoriously hard to test.

Currently, this area of operations and development feels very ripe for innovation.  We need a Ruby on Rails-style foundation to
make writing systems like this easy, but we don't even have established practices and techniques to codify in a framework yet.
Which means, we need people creating complex systems using these building blocks to learn.  Who is going to take that risk?
