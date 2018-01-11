Let's keep going with our budding event-based architecture.  Suppose that both `js/server.js` and the event bus are provided to
us by our cloud services provider.  Suppose our provider guarantees it will fire a `get` or `post` event when someone
makes an HTTP request to a server it's running on our behalf.  We could then imagine that our entire system is describable by a configuration file:

!CREATE_FILE js/app.js
const renderPage       = require("./renderPage.js");
const storeInDatabase  = require("./storeInDatabase.js");
const sendWelcomeEmail = require("./sendWelcomeEmail.js");

module.exports = {
  "get": "pageRequested", // fire pageRequested if get is fired
  "post": "emailSignup",  // fire emailSignup if post is fired
  "emailSignup": [
    storeInDatabase, // if emailSignup is fired, call storeInDatabase
    renderPage,      // if emailSignup is fired, ALSO call renderPage
  ],
  "pageRequested": [
    renderPage
  ],
  "newEmailAddress": [
    sendWelcomeEmail
  ]
}
!END CREATE_FILE

To keep our example actually executable locally, we'll modify `js/server.js` to read our new `js/app.js`.  Just remember, the
idea is that our cloud services provider would handle this plumbing and we'd simply provide `js/app.js`, so bear with me.

To mimic what our cloud services provider would do, we'll iterate over the configuration from `js/app.js` and connect events and
listeners and fire those events to the listeners at the right time.

!EDIT_FILE js/server.js /* */
{
  "match": "const qs   = require(\"querystring\");",
  "insert_after": [
    "const EventBus = require(\"./EventBus.js\");"
  ]
},
{
  "match": "const port     = 3000;",
  "insert_after": [
    "for (var eventName in app) {",
    "  if (app.hasOwnProperty(eventName)) {",
    "    const functions = app[eventName];",
    "    if (typeof functions === \"string\") {",
    "      EventBus.register(eventName, (params) => {",
    "        EventBus.fire(functions,params);",
    "      });",
    "    }",
    "    else {",
    "      functions.forEach( (f) => {",
    "        EventBus.register(eventName, f);",
    "      });",
    "    }",
    "  }",
    "}"
  ]
},
{
  "match": "      app(write,qs.parse(body));",
  "replace_with": [
    "      EventBus.fire(\"post\",{ write: write, body: qs.parse(body) });"
  ]
},
{
  "match": "    app(write);",
  "replace_with": [
    "    EventBus.fire(\"get\",{ write: write });"
  ]
}
!END EDIT_FILE

Restarting everything, it all still works:

!SHBG{pause=2} node js/server.js

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up4.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

Our log messages prove that this is happening:

!STOPBG{output=true} node js/server.js

We are now simulating events that a cloud services provider like AWS would fire, including:

* HTTP requests
* Infrastructure events like databases being modified
* Custom events we generate

<aside class="pullquote">The fundamentals of an object-oriented system are to send messages to objects that either do work, send more messages, or both.  That's what this is.</aside>

With these building blocks, we could completely describe a highly complex system.  If you are familiar with object-oriented
design and the definition of OO, this might seem familiar. The fundamentals of an object-oriented system are to send messages to
objects that either do work, send more messages, or both.  That's what this is.


Let's use these building blocks to enhance our system without changing anything that exists already.

## Open for Extension, Closed for Modification

Let's suppose our data science team wants to do some analysis on the domain names of email addresses
signed up for our site.  We can make this happen without changing any of the existing code.

First, we'll define a new function in `js/emailDeepLearning.js` that can be notified about new email addresses:

!CREATE_FILE js/emailDeepLearning.js
const addEmailToDataWarehouse = (data) => {
  const emailData = data["email"]["data"];
  console.log(`Storing ${JSON.stringify(emailData)} in our data warehouse for some deep learning!`);
}
module.exports = addEmailToDataWarehouse;
!END CREATE_FILE

We can hook that into our system by adding it to the configuration in js/app.js:

!EDIT_FILE js/app.js /* */
{
  "match": "const sendWelcomeEmail = require(\"./sendWelcomeEmail.js\");",
  "insert_after": [
    "const addEmailToDataWarehouse = require(\"./emailDeepLearning.js\");"
  ]
},
{
  "match": "    sendWelcomeEmail",
  "replace_with": [
    "    sendWelcomeEmail,",
    "    addEmailToDataWarehouse"
  ]
}
!END EDIT_FILE

And, restarting the server and trying it out, we can see that all our existing code still runs, but we're now also applying the latest and greatest Big Data Analysis techniques on the newly-added email.

!SHBG{pause=2} node js/server.js

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up5.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

!STOPBG{output=true} node js/server.js

Notice that we didn't change *any* of the existing code.  If you have a better example of the oft-confused [open/closed
principle](https://en.wikipedia.org/wiki/Open/closed_principle), I'd like to see it.

Also notice that our data science team got what it needed without having to know about the underlying database schema.  Traditionally, business intelligence teams would pull from a backup or read replica, which couples them to that schema, making it harder to change (or worse, affecting production workloads).  That's not possible here—by design.  All we have to do is make sure the schema of the *messages* stays the same (which we'll talk about more in a few chapters).

Now, take these concepts and think about a more realistic large system, evolved over years.  It's impossible for one person to
understand such systems—they are too large.  If said system is built using decoupled message-based interactions, it should be
much harder to break and thus easier to change.  We demonstrated that just now—the new function for the data science team would
not be able to break the core functions already implemented.

Aside from the programming model, there are operational advantages to this as well.
