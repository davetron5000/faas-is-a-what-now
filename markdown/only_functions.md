Let's keep going with our budding event-based architecture.  Suppose that both `js/server.js` and the event bus are provided to
us for us by our cloud services provider.  Suppose our provider guarantees it will fire a `get` or `post` event when someone
makes an HTTP request to a server it's running on our behalf.  We could then imagine that our entire system is describable by a configuration file:

!CREATE_FILE js/app.js
const renderPage       = require("./renderPage.js");
const storeInDatabase  = require("./storeInDatabase.js");
const sendWelcomeEmail = require("./sendWelcomeEmail.js");

module.exports = {
  "get": "pageRequested", // fire pageRequested if get is fired
  "post": "emailSignup",  // fire emailSignup is post is fired
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

Although our cloud services provider would be capable of reading `js/app.js` to run our system, we aren't using one in these
examples, so let's modify `js/server.js` to handle this new configuration format.  Remember, this isn't code you would normally
write, but it's important to see these examples running.

We'll keep it as simple as we can.  We need to iterate over the structure exported by `js/app.js` and wire up all the events and
listeners.  We then need our web server to fire the `get` and `post` events, instead of calling `app(write)`.

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

If we supposed that our cloud services provider can:

* Send events for HTTP requests
* Send events for infrastructure events like databases being modified
* Send arbitrary events we generate

We could completely describe a highly complex system using just events and functions.  We could also enhance this system in a
more safe way.  For example, suppose our data science team wanted to do some analysis on the domain names of email addresses
signed up for our site.  We could do this without changing the basic logic that we have now by configuring a new function:

!CREATE_FILE js/emailDeepLearning.js
const addEmailToDataWarehouse = (data) => {
  const emailData = data["email"]["data"];
  console.log(`Storing ${JSON.stringify(emailData)} in our data warehouse for some deep learning!`);
}
module.exports = addEmailToDataWarehouse;
!END CREATE_FILE

And then adding that to our system's manifest:

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

And, restarting the server and trying it out, we can see that we're now applying the latest and greatest Big Data Analysis
techniques on the newly-added email.

!SHBG{pause=2} node js/server.js

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up5.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

!STOPBG{output=true} node js/server.js

It's also worth pointing out that the traditional way to do Business Intelligence like this is to allow the data scientists to
reach into our databases directly.  This couples them to our schema and presents operational challenges (e.g. if they run an
inefficient query against our production system).  By using events, they are totally decoupled and we can evolve our systems
independently—as long as we continue to send the messages in the right way (which we'll talk about more in a few chapters).

If you imagine this way of working, and apply it to a large system where no one person can understand how it all works, and where
various teams of engineers are responsible for different business functions, this has a lot of advantages.  In our example above,
it's actually pretty difficult to break the email-sign-up flow by adding more logic to it.  We could add many other
functions to this flow without ever having to worry about the main flow breaking.

Aside from the programming model, there are operational advantages to this as well.
