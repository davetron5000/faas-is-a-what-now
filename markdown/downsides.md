By designing our system as a series of functions that react to events, we've seen a lot of advantages, from the programming to the operational model, and it might even be cheaper to run, at least if all we care about is our AWS bill (sidenote: mature organizations aren't this myopic).

There are some downsides that we can very obviously see in just the simple system that we created.

The first is that for a simple system, it feels overcomplex.  We haven't talked about how we might manage deployment or how to
run this in our development environment. Frameworks like [the Serverless Framework](https://serverless.com) provide solutions for
this, but this is not without complexity.  Especially if you have years of battle-hardened experience writing and deploying more
conventional web apps—you've got a learning curve ahead, and it's not gentle.

<aside class="pullquote">You've got a learning curve ahead, and it's not gentle</aside>

A moderately involved system is likely also to be complex if you don't make good design choices.  Ideally, new features (or changes) could be localized to one or a few functions.  This would only be true if you'd made the right choices around what functions did what, and set proper system boundaries.  In the case where you didn't, you would end up needing to make a lot of changes in a lot of functions.


In a monolithic system, development tools and workflows make this as simple as can be expected.  Be it text editor or IDE,
navigating around a codebase is a well-solved problem.  When your codebase is now spread over many disparate functions,
developer workflows will have to change, and might be generally more difficult.

Of course, this brings us to the biggest downside, which is…how are we supposed to test all this?

## Unit Tests Don't Always Inspire Confidence

Let's write some unit tests real quick.  We won't set up a test framework or anything.  Instead, we'll create some files that
exercise our functions and blow up if anything's wrong.

First, a test for `renderPage` in `js/renderPageTest.js`.  We'll create our own version of `write` that stores what it's given,
call `renderPage`, then see if it wrote out something reasonable.

!CREATE_FILE js/renderPageTest.js
const renderPage = require("./renderPage.js")

var stringWritten = null;

const write = function(string) {
  stringWritten = string;
}

renderPage({ write: write, body: { email: "pat@example.com" } });
try {
  if (!stringWritten.match(/\<strong\>\d+ startups/)) {
    throw "Could not find count";
  }
  if (!stringWritten.match(/As of \<strong\>/)) {
    throw "Could not find date";
  }
  if (stringWritten.indexOf("Thanks pat@example.com") === -1) {
    throw "Couldn't find 'Thanks pat@example.com' on the page";
  }
  console.log("✅ renderPage is good");
} catch (exception) {
  console.log(`🚫 renderPage is broken: ${exception}`);
}
!END CREATE_FILE

Next up, a test for `storeInDatabase`, which we'll put in `js/storeInDatabaseTest.js`

!CREATE_FILE js/storeInDatabaseTest.js
const storeInDatabase = require("./storeInDatabase.js");
const Database = require("./Database.js");

storeInDatabase({ body: { email: "pat@example.com" }});

var found = false;
for (id in Database.data) {
  if (Database.data[id] === "pat@example.com") {
    found = true;
  }
}

if (found) {
  console.log("✅ storeInDatabase is good");
}
else {
  console.log("🚫 storeInDatabase didn't save pat@example.com");
}
!END CREATE_FILE

Finally, we'll create a test for `sendWelcomeEmail`.  Since our implementation returns the email, we'll call the function and
make sure the right email is returned.  This isn't a great test, but our implementation just prints stuff to the console, so it's
better than nothing.  It'll go in `js/sendWelcomeEmailTest.js`

!CREATE_FILE js/sendWelcomeEmailTest.js
const sendWelcomeEmail = require("./sendWelcomeEmail.js")

const data = {
  email: {
    data: {
      id: 12,
      email: "pat@example.com"
    }
  }
}

const emailMailed = sendWelcomeEmail(data);
if (emailMailed === "pat@example.com") {
  console.log("✅ sendWelcomeEmail is good");
}
else {
  console.log(`🚫 sendWelcomeEmail expected pat@example.com, got ${emailMailed}`);
}
!END CREATE_FILE

Now, we can run our tests one at a time, using `node`:

!SH node js/renderPageTest.js
!SH node js/storeInDatabaseTest.js
!SH node js/sendWelcomeEmailTest.js

What's missing here?  A test of the entire system.  Although we created `server.js` and `EventBus.js` for demonstration purposes,
remember in the real world, these are provided by our cloud services provider and so we can't exactly run all of AWS on our
laptop to execute this entire system.  And even if we *could*, our system might be so complex that this is infeasible.

We can see the downsides of this by changing `renderPage`.  Let's say instead of expecting `email` to be in the `params.body`, we
expect the key to be `emailAddress`:

!EDIT_FILE js/renderPage.js /* */
{
  "match": "  if (params.body && params.body.email) {",
  "replace_with": [
    "  if (params.body && params.body.emailAddress) {"
  ]
},
{
  "match": "    html = html.replace(\"##email##\",`Thanks ${params.body.email}`);",
  "replace_with": [
    "    html = html.replace(\"##email##\",",
    "                        `Thanks ${params.body.emailAddress}`);"
  ]

}
!END EDIT_FILE

Our unit test will fail:

!SH node js/renderPageTest.js

But, we can fix it by changing the param name we use in the test:

!EDIT_FILE js/renderPageTest.js /* */
{
  "match": "renderPage({ write: write, body: { email: \"pat@example.com\" } });",
  "replace_with": [
    "renderPage({ write: write, body: { emailAddress: \"pat@example.com\" } });"
  ]
}
!END EDIT_FILE

Now, our tests pass again:

!SH node js/renderPageTest.js

But, our system is horribly broken, since the HTML page is still submitting `email` as the parameter name.  Without a system test, we might not even know that things aren't working.

## System Testing for Faas is Not Yet a Thing

The Serverless Framework's [testing page](https://serverless.com/framework/docs/providers/aws/guide/testing/) is a bit hand-wavy.
It's hard to blame them, because this is hard.  It's hard for microservices, and it's hard for Faas-based systems as well. This
is where a tightly-integrated, monolithic system actually shines.  You actually *can* stand up the world and run tests against
it.

The way this is solved in a microservices-based architecture is to use [_consumer-driven-contracts_](https://martinfowler.com/articles/consumerDrivenContracts.html).  A consumer-driven contract
allows the consumer of a microservice to record assertions about how it expects that service to behave.  For example, we might
send a `GET` request to `payments.example.com/credit_cards/42` to get user 42's payment details and expect some JSON back.  As a consumer of `payments.example.com`, we'd record that when we submit that request, we expect a certain response that matches a certain format:

```json
{
  "description": "Getting payment details",
  "request": {
    "method": "GET",
    "url": "/credit_cards/42"
  },
  "response": {
    "credit_card": {
      "token": "asfasfgadsf",
      "type": "visa",
      "lastFour": 1234,
      "expiration": "5/19"
    }
  }
}
```

We then *publish* this expectation to the service itself.  Call this a _contract_.  The service takes this contract and executes it *against itself*.  If it behaves as the consumer expects, we have confidence that these two systems are talking to each other
properly and that they will work in production.

Consumer-driven contracts aren't a household name with developers, but the concept is well established and tooling exists to set
this up without a lot of trouble.

This concept is non-existent for event-sourced systems like the one we've built.

*This* is a pretty big downside.  You could address this by adopting a testing-in-production mindset, where all changes are
hidden under a feature flag and you arrange for test data in production, so you can deploy all changes and evaluate them against
the real systems.  This is a total mindshift and really bleeding edge, but it's a legit way to do this.

Even if you *did* do this, it's a helluva long feedback cycle.  It would be more ideal to have something to prevent deploys that
would break production or some way to get confidence in your code during development.

Could we do consumer-driven contracts with event-sourcing and messaging?
