Let's keep going with our budding event-based architecture.  Suppose that both `js/server.js` and the event bus are provided to
us for us to use.  We could then imagine that our entire sytem is describable by a configuration file:

!CREATE_FILE js/app.js
const renderPage       = require("./renderPage.js");
const storeInDatabase  = require("./storeInDatabase.js");
const sendWelcomeEmail = require("./sendWelcomeEmail.js");

module.exports = {
  // This means "on the emailSignup event, fire both
  // storeInDatabase and renderPage
  "emailSignup": [
    storeInDatabase,
    renderPage,
  ],
  "pageRequested": [
    renderPage
  ],
  "newEmailAddress": [
    sendWelcomeEmail
  ],
  // This means "re-fire the post event as emailSignup"
  "post": "emailSignup",
  // This means "re-fire the vet event as pageRequested"
  "get": "pageRequested"
}
!END CREATE_FILE

To make this work, we beef up `js/server.js` (remember that this isn't code we'd have to write ourselves, so don't worry too
much about it—it's here to we can see our code working):

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
singed up for our site.  We could do this without changing the basic logic that we have now by configuring a new function:

!CREATE_FILE js/emailDeepLearning.js
const Database = require("./Database.js");

const addEmailToDataWarehouse = (emailId) => {
  const email = Database.find(emailId);
  console.log(`Storing ${email} in our data warehouse for some deep learning!`);
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

If you imagine this way of working, and apply it to a large system where no one person understands how it all works, and where
various teams of engineers are responsible for different business functions, this has a lot of advantages.  In our example above,
it's actually pretty difficult to break the email-sign-up flow by adding more logic to it.  We could add many other
functions to this flow without ever having to worry about the main flow breaking.

Aside from the programming model, there are operational advantages to this as well.
