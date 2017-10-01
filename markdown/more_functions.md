I promise we'll get to FaaS and serverless, but we have to see something more real than serving up a static page.  Let's add a
couple of features to our app.

Let's add a form, so that users can submit their email address to learn about our amazing website.  We'll save that to a database
and launch a background job that sends them a welcome email.  We're not going to set up a real database or background processing
system here, but we'll write code as if we have.

## A More Realistic System

First, let's change `js/server.js`, as well as our function protocol to allow posting data.  We'll keep it simple: if the HTTP
method is a POST, we'll parse the body and send it to the function as an argument.

!EDIT_FILE js/server.js /* */
{
  "match": "const http = require(\"http\");",
  "insert_after": [
    "const qs   = require(\"querystring\");"
  ]
},
{
  "match": "  app(write);",
  "replace_with": [
    "  if (req.method == \"POST\") {",
    "    let body = [];",
    "    req.on('data', (chunk) => {",
    "      body.push(chunk);",
    "    }).on('end', () => {",
    "      body = Buffer.concat(body).toString();",
    "      app(write,qs.parse(body));",
    "    });",
    "  }",
    "  else {",
    "    app(write);",
    "  }"
  ]
}
!END EDIT_FILE

This is mostly boilerplate Node stuff, so nothing revolutionary here. We need to change `js/app.js`, but first we need to add a signup form to our template and, while we're there, let's also add a place to render a confirmation
message.

!EDIT_FILE html/index.html <!-- -->
{
  "match": "  <body>",
  "insert_after": [
    "    <h3>##email##</h3>"
  ]
},
{
  "match": "  </body>",
  "insert_before": [
    "    <form action=\"/\" method=\"post\">",
    "      <label>",
    "        Sign Up",
    "      </label>",
    "      <input type=\"email\" name=\"email\" />",
    "    </form>"
  ]
}
!END EDIT_FILE

Now, we'll replace `js/app.js` with a version that handles the POST.  I promise this is leading to something.

!CREATE_FILE js/app.js
const fs   = require("fs");
const path = require("path");
const index = fs.readFileSync(
      path.resolve(__dirname,"..","html","index.html")
    ).toString();

const renderPage = (write,params) => {
  var html = index.replace("##date##",(new Date()).toString()).
    replace("##count##",Math.floor(Math.random() * 1000));

  if (params && params.email) {
    html = html.replace("##email##",`Thanks ${params.email}`);
  }
  else {
    html = html.replace("##email##","");
  }
  write(html);
}

const emailSignup = (write,params) => {
  console.log(`Signing up ${params.email}`);
  renderPage(write,params);
}

module.exports = (write,body) => {
  if (body) {
    emailSignup(write,body);
  }
  else {
    renderPage(write);
  }
}
!END CREATE_FILE

It's a bit of a slog, but `renderPage` is our rendering function, and `emailSignup` handles the POST, which is just outputting to
the console for now.

If we start our server, the app is still working, but rendering our sign-up form:

!SHBG{pause=2} node js/server.js

!SCREENSHOT "Updated App with Sign-up Form" http://localhost:3000 sign_up.png

If we submit the form, it works and re-renders the page:

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

!STOPBG node js/server.js

OK, so what has this to do with events and function?  One more step before we're done.  We want to store the email address in a database and then fire off a background job to send a welcome email to that address.

We'll fake this out by creating some classes.  First, we'll make a very cheesy database:

!CREATE_FILE js/Database.js
class Database {
  constructor() {
    this.data = {};
    this.nextId = 1;
  }
  save(email) {
    const id = this.nextId;
    this.nextId++;

    this.data[id] = email;
    return id;
  }
  find(id) {
    return this.data[id];
  }
}
module.exports = (new Database());
!END CREATE_FILE

We'll also make a cheesy background job system that just executes a function after 500ms:

!CREATE_FILE js/BackgroundJob.js
class BackgroundJob {
  constructor(f) {
    this.job = f;
  }
  performLater(args) {
    setTimeout(() => { this.job(args) },500);
  }
}
module.exports = BackgroundJob
!END CREATE_FILE

Now, we'll require them in `js/app.js`:

!EDIT_FILE js/app.js /* */
{
  "match": "    ).toString();",
  "insert_after": [
    "",
    "const Database      = require(\"./Database.js\");",
    "const BackgroundJob = require(\"./BackgroundJob.js\");"
  ]
}
!END EDIT_FILE

Armed with this powerful array of webscale subsystems, we can enhance `emailSignup` to save the email and queue the background job.

!EDIT_FILE js/app.js /* */
{
  "match": "  console.log(`Signing up ${params.email}`);",
  "insert_after": [
    "  const backgroundJob = new BackgroundJob((emailId) => {",
    "    const email = Database.find(emailId);",
    "    console.log(`Sending welcome email to ${email}`);",
    "  });",
    "  const id = Database.save(params.email);",
    "  console.log(`${params.email} saved to the DB with id ${id}`);",
    "  backgroundJob.performLater(id);"
  ]
}
!END EDIT_FILE

Restart the server:

!SHBG{pause=2} node js/server.js

Submit the form, which should still work fine:

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up2.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

And we should see our log messages:

!STOPBG{output=true} node js/server.js

With that in place, we have a more complex system to reason about, so let's take a fresh look at it as a series of events.

## In Which Events Become More Clear

Although our code is very procedural, the overall processes we're implementing aren't as coupled as it would seem.  If we were to look at our system through the lens of causes and effects, we might find it looks something like
this:

!GRAPHVIZ event_based.png Our system as a series of events
digraph faas {

  nodesep=0.5
  rankdir="LR"

  node[fontname="Avenir" fontsize="16" margin="0.4,0.3"]
  edge[fontname="Avenir" fontsize="16"]

  SubmitsEmail         [ shape="rarrow"]
  RenderPage           [ shape="rectangle"]
  StoreInDatabase      [ shape="rectangle"]
  NewEmailAddressAdded [ shape="rarrow"]
  RequestsPage         [ shape="rarrow"]
  SendWelcomeEmail     [ shape="rectangle"]

  User                 -> SubmitsEmail
  User                 -> RequestsPage
  RequestsPage         -> RenderPage
  SubmitsEmail         -> RenderPage
  SubmitsEmail         -> StoreInDatabase
  StoreInDatabase      -> NewEmailAddressAdded
  NewEmailAddressAdded -> SendWelcomeEmail

}
!END GRAPHVIZ

The arrows in this diagram are _events_, and the boxes are functions that get called based on those events.  This shows the logical flow of what is triggering what, but omits all of the technical details (for example, this doesn't mention background jobs since that's an implementation detail—not directly relevant to the problem).

Let's re-design our fledgling system to be based around events and reactions to those events.

First, let's create functions for each of the steps.  We'll need a function that renders the page (which we largely have already in `renderPage`), one to store an email address in the database, and one to send a welcome email.

Instead of doing that inline, we'll make files for these.  First up, `renderPage`, which is almost identical to the current function.  One change we'll make is to replace the explicit params with a general `params` object.  We'll
expect to find `params.write` in it, and optionally `params.body`:

!CREATE_FILE js/renderPage.js
const fs   = require("fs");
const path = require("path");
const index = fs.readFileSync(
      path.resolve(__dirname,"..","html","index.html")
    ).toString();

const renderPage = (params) => {
  var html = index.replace("##date##",(new Date()).toString()).
    replace("##count##",Math.floor(Math.random() * 1000));

  if (params.body && params.body.email) {
    html = html.replace("##email##",`Thanks ${params.body.email}`);
  }
  else {
    html = html.replace("##email##","");
  }
  params.write(html);
}
module.exports = renderPage;
!END CREATE_FILE

Next, we need one last function to store the email in the database.

!CREATE_FILE js/storeInDatabase.js
const Database      = require("./Database.js");
const storeInDatabase = (params) => {
  const email = params.body.email;
  console.log(`Signing up ${email}`);
  const id = Database.save(email);
  console.log(`${email} saved to the DB with id ${id}`);
}
module.exports = storeInDatabase;
!END CREATE_FILE

Lastly, we'll create a function that sends a welcome email (which we hand-wave over).

!CREATE_FILE js/sendWelcomeEmail.js

const sendWelcomeEmail = (data) => {
  const email = data["email"]["data"]["email"];
  console.log(`Sending welcome email to ${email}`);
  return email;
}
module.exports = sendWelcomeEmail;
!END CREATE_FILE

To make our system work, we need to arrange for these functions to respond to the right events.  This means we need some way to fire events and register these functions as reactors to those events.

Let's create a simple event bus class to do that:

!CREATE_FILE js/EventBus.js
class EventBus {
  constructor() {
    this.eventListeners = {};
  }
  /**
   * Register a listener with an arbitrary event.
   *
   * @param {string} eventName - the name of the event.  Can be anything.
   * @param {function} listener - the event listener.  Should be a function
   *                              and will be given whatever
   *                              params were given to fire
   */
  register(eventName,listener) {
    if (!this.eventListeners[eventName]) {
      this.eventListeners[eventName] = [];
    }
    this.eventListeners[eventName].push(listener);
  }
  /**
   * Fire an event and notify any listeners.

   * @param {string} eventName - name of the event.
   * @param {*} params - an object representing the parameters
                         relevant to the event.
   */
  fire(eventName,params) {
    if (!this.eventListeners[eventName]) {
      console.log(`No listeners for ${eventName}`);
      return;
    }
    this.eventListeners[eventName].forEach( (listener) => {
      listener(params);
    })
  }
}
module.exports = (new EventBus());
!END CREATE_FILE

This isn't anything special, it just stores listeners on events and provides a way to fire events to notify those listeners.

We will need to modify our `Database` class to fire an event when it saves an email.

!EDIT_FILE js/Database.js /* */
{
  "match": "class Database {",
  "insert_before": [
    "const EventBus = require(\"./EventBus.js\");",
    ""
  ]
},
{
  "match": "    return id;",
  "insert_before": [
    "    EventBus.fire(\"newEmailAddress\",{ email: { action: \"created\", data: { id: id, email: email }}});"
  ]
}
!END EDIT_FILE

With these pieces, `js/app.js` becomes not much more than configuration:

!CREATE_FILE js/app.js
const EventBus         = require("./EventBus.js");
const renderPage       = require("./renderPage.js");
const storeInDatabase  = require("./storeInDatabase.js");
const sendWelcomeEmail = require("./sendWelcomeEmail.js");

EventBus.register("emailSignup"     , storeInDatabase);
EventBus.register("emailSignup"     , renderPage);
EventBus.register("pageRequested"   , renderPage);
EventBus.register("newEmailAddress" , sendWelcomeEmail);

module.exports = (write,body) => {
  if (body) {
    EventBus.fire("emailSignup",{ write: write, body: body});
  }
  else {
    EventBus.fire("pageRequested", { write: write });
  }
}
!END CREATE_FILE

If we restart our server, navigate to the page, and submit our email, everything still works:

!SHBG{pause=2} node js/server.js

!DO_AND_SCREENSHOT "Signed Up" http://localhost:3000 signed_up3.png
document.getElementsByName("email")[0].value = "pat@example.com";
document.getElementsByTagName("form")[0].submit();
!END DO_AND_SCREENSHOT

Our log messages prove that this is happening:

!STOPBG{output=true} node js/server.js

So, now what?  There's a lot going on in what we just did.  Certainly, functional decomposition and putting stuff in different
files helps keep our code organized, but all the logic of what happens during sign up is gone!

What we've done has some nice benefits:

* The code that stores an email in a database no longer has to remember to render a page.  Instead, we've connected *two*
listeners to the same event—`emailSignup`—one which renders the page and one which does the signup.
* The orchestration code is separate from the code for handling events, meaning each could be tested much more simply and each is
easier to understand.
* We no longer have to mess with background jobs.

This does have a big downside, which is that it's now pretty difficult to piece together what happens when a user submits their
email addressto us.  *But*, I'd argue that from the configuration that is now inside `js/app.js`, this information could be
mechanically derived.

So, what about serverless architecture and FaaS?

## What if you don't owne orchestration?

When deploying a system that runs inside a web server like Node, or even a more sophisticated framework like Ruby on Rails, you
still end up with a lot of what is essentially configuration and orchestration—getting all the little pieces to work together.

In our event-based system that has gone away.  We have two bits of code (in `js/server.js` and `js/app.js`) that do nothing but
assemble our app together.  What if, instead of having to write that plumbing ourselves, it was given to us by, say, a cloud
services provider?
