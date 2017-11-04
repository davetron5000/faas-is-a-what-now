A big downside of designing a distributed system built on Faas it that it's hard to execute system tests or to even test that
various functions are working together properly.  In a microservices world, we can use consumer-driven contracts, which we talked
about in the last chapter.

What is the analog in our event-sourced, functions-as-a-service, serverless world?

Since all activity is based on messages, the way we'd test this is to use these messages as our test input, and then to assure
that the messages actually are produced in the way our code thinks.

In the previous section we broke our system by changing the format of the message the `renderPage` function received.

Let's change how we test to be more message focused.  We'll start by using the documented messages and payloads as input to our
tests, rather than hand-coding them.

## Output of One is Input to Another

The problem with our system as it stands is that `renderPage` is expecting a message that won't be sent. If we'd been more
disciplined about our test data from the start, we might not have caused this problem.

Let's create an event catalog that describes all the events we know about.  We'll create a directory named `js/test_messages` and
place one file in there for each event.  The contents of the file will be an example event.

First, we'll create `js/test_messages/emailSignup.json`:

!CREATE_FILE js/test_messages/emailSignup.json
{
  "body": {
    "email": "pat@example.com"
  }
}
!END CREATE_FILE

Next, `js/test_messages/newEmailAddress.json`:

!CREATE_FILE js/test_messages/newEmailAddress.json
{
  "email": {
    "data": {
      "id": 42,
      "email": "pat@example.com"
    }
  }
}
!END CREATE_FILE

And finally, `js/test_messages/pageRequested.json`:

!CREATE_FILE js/test_messages/pageRequested.json
{}
!END CREATE_FILE

We'll then write an event catalog in `js/EventTestCatalog.js` to read these in.

!CREATE_FILE js/EventTestCatalog.js
const fs   = require("fs");
const path = require("path");

const EventTestCatalog = {
  events: {}
};

const testMessagesPath = path.resolve(__dirname,"test_messages");
fs.readdirSync(testMessagesPath).forEach(file => {
  const message = JSON.parse(
                    fs.readFileSync(
                      path.resolve(testMessagesPath,file)
                    ));
  const eventName = file.replace(/\.json$/,"");
  EventTestCatalog.events[eventName] = message;
});


EventTestCatalog.events["get"] = EventTestCatalog.events["pageRequested"];
EventTestCatalog.events["post"] = EventTestCatalog.events["emailSignup"];

module.exports = EventTestCatalog;
!END CREATE_FILE

Now that we've created some canonical message bodies for our messages outside of any given function implementation, let's rewrite
`js/renderPageTest.js` to use these messages as input.

Because  `renderPage` is configured to handle both `pageRequested` and `emailSignup`, we'll execute our tests twice, one for each.

!CREATE_FILE js/renderPageTest.js
const renderPage = require("./renderPage.js")
const EventTestCatalog = require("./EventTestCatalog.js");

var stringWritten = null;

const write = function(string) {
  stringWritten = string;
}

const events = ["pageRequested","emailSignup"];
events.forEach( (eventName) => {
  try {
    const testMessage = EventTestCatalog.events[eventName];
    testMessage.write = write;
    renderPage(testMessage);

    if (stringWritten.match(/\<strong\>\d+ startups/)) {
      if (stringWritten.match(/As of \<strong\>/)) {
        if (eventName === "emailSignup") {
          if (stringWritten.indexOf("Thanks pat@example.com") === -1) {
            throw "couldn't find 'Thanks pat@example.com' on the page";
          }
        }
      }
      else {
        throw "could not find date";
      }
    }
    else {
      throw "could not find count";
    }
    console.log(`✅ given ${eventName}, renderPage is good`);
  }
  catch (exception) {
    console.log(`🚫 given ${eventName}, renderPage is broken: ${exception}`);
  }
});
!END CREATE_FILE

Now, when we run our test, we get a failure:

!SH node js/renderPageTest.js

We can see that our code works for the `pageRequested` event, but not for the `emailSignup` one.  Nice! This gives us a better
idea of where things are broken.

A brief investigation shows that we're expecting `emailSignup` and not `email`, so we can make the test pass by fixing the
`renderPage` function:

!EDIT_FILE js/renderPage.js /* */
{
  "match": "  if (params.body && params.body.emailAddress) {",
  "replace_with": [
    "  if (params.body && params.body.email) {"
  ]
},
{
  "match": "    html = html.replace(\"##email##\",`Thanks ${params.body.emailAddress}`);",
  "replace_with": [
    "    html = html.replace(\"##email##\",`Thanks ${params.body.email}`);"
  ]
}
!END EDIT_FILE

Our tests should now pass:

!SH node js/renderPageTest.js

What does this mean?

## Messages as Integration Test Data

This means that if our functions always take messages, and we clearly document the sorts of messages they accept, we can use
actual messages as test input to drive our functions.

It *also* means that we can record any messages *sent* by our code to feed into this centralized system.  Consider
`sendWelcomeEmail`.  It accepts a somewhat complex structure as input.  As we did in `renderPageTest.js`, let's remove the
hard-coded test data and grab a message from our  `EventTestCatalog`:

!CREATE_FILE js/sendWelcomeEmailTest.js
const sendWelcomeEmail = require("./sendWelcomeEmail.js")
const EventTestCatalog = require("./EventTestCatalog.js");

const data = EventTestCatalog.events["newEmailAddress"];

const emailMailed = sendWelcomeEmail(data);
if (emailMailed === "pat@example.com") {
  console.log("✅ sendWelcomeEmail is good");
}
else {
  console.log(`🚫 sendWelcomeEmail is failed.  Expected pat@example.com, got ${emailMailed}`);
}
!END CREATE_FILE

Our test still passes:

!SH node js/sendWelcomeEmailTest.js

OK, that was a repeat of the above, but let's suppose that we have a test of our database that actually outputs the message to
our catalog?

First, we'll add a function to `EventTestCatalog` to allow saving new message payloads:

!EDIT_FILE js/EventTestCatalog.js /* */
{
  "match": "  events: {}",
  "replace_with": [
    "  events: {},",
    "  saveEvent: (name,body) => {",
    "    fs.writeFileSync(path.resolve(testMessagesPath,`${name}.json`),JSON.stringify(body,null,2))",
    "  }"
  ]

}
!END EDIT_FILE

Now, our test of `Database.js` will update the test message if its tests pass:

!CREATE_FILE js/DatabaseTest.js
const Database = require("./Database.js");
const EventBus = require("./EventBus.js");
const EventTestCatalog = require("./EventTestCatalog.js");

/* store the messages we got by subscribing
   to the event we expect to be fired */
var receivedMessage = null;
const listener = (data) => {
  receivedMessage = data;
}
EventBus.register("newEmailAddress",listener)

try {
  const savedId = Database.save("pat@example.com");

  if (Database.find(savedId) !== "pat@example.com") {
    throw "save & find FAILED";
  }

  if (receivedMessage === null) {
    throw "Events did not fire";
  } else if (receivedMessage.email.data.id  !== savedId) {
    throw `Expected ${savedId}, but got ${receivedEmailId}`;
  }
  console.log("✅ Database is good");

  // Save the message we received to the shared test catalog
  EventTestCatalog.saveEvent("newEmailAddress",receivedMessage);
} catch (exception) {
  console.log(`🚫 Database failed: ${exception}`);
}
!END CREATE_FILE

We'll run the test:

!SH node js/DatabaseTest.js

And we can see that the test message for `newEmailAddress` has changed:

!SH cat js/test_messages/newEmailAddress.json

If we re-run `js/sendWelcomeEmailTest.js`, it will read this file, and should still pass:

!SH node js/sendWelcomeEmailTest.js

So far so good.   Now, let's say the database changes its payload format for the messages, by changing the field `email` to
`emailAddress`:

!EDIT_FILE js/Database.js /* */
{
  "match": "    EventBus.fire(\"newEmailAddress\",{ email: { action: \"created\", data: { id: id, email: email }}});",
  "replace_with": [
    "    EventBus.fire(\"newEmailAddress\",{ email: { action: \"created\", data: { id: id, emailAddress: email }}});"
  ]
}
!END EDIT_FILE

The test for the database still passes:

!SH node js/DatabaseTest.js

Since it does, it outputs an updated test message:

!SH cat js/test_messages/newEmailAddress.json

Which now breaks `sendWelcomeEmail`:

!SH node js/sendWelcomeEmailTest.js

Nice!

## What Does it Mean?

What this demonstrates is that it's possible to create a system based on events and functions and, with sufficient investment in
how we write tests, we can achieve the level of confidence we'd get from an end-to-end test by keeping track of the messages and
expectations of the various functions and what they do.

You could imagine taking the above concept to the next level by having a central database of messages, and a registry of
functions that send those messages along with those that expect to receive them.  It would then be possible to examine this
registory to see if anything broke.  For example, if we'd made the change to `Database` above under such a sytem, we could
include, as part of our CI process, a step that asks the central registory “Will this updated payload break any registered
consumers?”.

Such a system could also monitor *all* messages and, armed with the knowledge of who is expecting what from whom, indicate if an
expected message stopped being sent, or if no one was registered to receive a particular message.

Neat!  So, we should all do serverless now?


