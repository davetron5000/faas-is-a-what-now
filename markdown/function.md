The building blocks of a FaaS-based system are _functions_ that get fed _events_.  Rather than go into some abstract example that calculates Fibonacci or some other terrible thing, let's start with a real example - serving a webpage.

## A Toy App

First, let's create an HTML file to serve up.  We'll put it in `html/` in our work directory:

!SH mkdir html

Our HTML will have some dynamic elements to it, just so we can see that actually serving it is having an effect and we aren't
just dumping a static page.  In this case, we'll invent a simple templating syntaxing using hashes:

!CREATE_FILE html/index.html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>The Serverless Compedium</title>
  </head>
  <body>
    <h1>The Serverless Compedium</h1>
    <h2>An exhaustive tour of ƒ-as-a-service</h2>
    <p>
      Welcome to the next level of abstraction and
      scalability! As of <strong>##date##</strong>,
      there are <strong>##count## startups</strong>
      built entirely on functions!
    </p>
  </body>
</html>
!END CREATE_FILE

Next, let's create a simple web server in Node to serve this page.  You'll need [to install
Node](https://nodejs.org/en/download/) to follow along.

!SH node --version

We'll put all our JavaScript in `js/`:

!SH mkdir js

Now, let's create the file to serve this up.  We'll need to read the template in first, and then each time we are asked to serve
a request, render the template, which we'll do using the sophisticated `String.replace` function:

!CREATE_FILE js/server.js
const http = require("http");
const fs   = require("fs");
const path = require("path");

const hostname = "127.0.0.1";
const port     = 3000;

const index = fs.readFileSync(
      path.resolve(__dirname,"..","html","index.html")
    ).toString();

const server = http.createServer((req, res) => {
  const html = index.
                 replace("##date##",(new Date()).toString()).
                 replace("##count##",Math.floor(Math.random() * 1000));
  res.statusCode = 200;
  res.setHeader("Content-Type", "text/html");
  res.end(html);
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
!END CREATE_FILE

We can start our server thusly:

!SHBG{pause=2} node js/server.js

Navigating to `http://localhost:3000`, we can see our dynamic web server in all its glory:

!SCREENSHOT "Dynamic Web Page" http://localhost:3000 served_page.png

!STOPBG node js/server.js

This is obviously a tiny example, but if you've built any sort of dynamic website before, you can see that this is a miniaturized
version of something that could be complex.

To deploy this to production, we would need somewhere to deploy it. Services like Heroku provide platforms to run Node apps, but
we could also provision virtual machines on AWS, or even set up Node with a real web server on our computer provided by a service
like Linode.  The more “close to the metal” we get, the more we have to deal with.  Heroku handles a lot of stuff for us, Linode
handles nothing.

Beyond hosting this app, look at the code.  Almost none of it is special to our use case and our app.  Only three lines of this
app are specific to our problem:

```javascript
const html = index.
               replace("##date##",(new Date()).toString()).
               replace("##count##",Math.floor(Math.random() * 1000));
```

Let's rewrite the app so that the different parts are more obviously separated.

## Separating Concerns

We'll put our stuff in a new file called
`js/app.js`:

!CREATE_FILE js/app.js
const fs   = require("fs");
const path = require("path");
const index = fs.readFileSync(
      path.resolve(__dirname,"..","html","index.html")
    ).toString();

module.exports = (write) => {
  write(index.replace("##date##",(new Date()).toString()).
              replace("##count##",Math.floor(Math.random() * 1000)));
}
!END CREATE_FILE

Instead of returning a string to render, we'll accept a function we can call that will render that for us.  This will become handy later, so trust me for now that this makes sense.

Now, `js/server.js` is simpler:

!CREATE_FILE js/server.js
const http = require("http");
const app  = require("./app");

const hostname = "127.0.0.1";
const port     = 3000;

const server = http.createServer((req, res) => {
  const write = (allData) => {
    res.write(allData, () => {
      res.end();
    });
  }
  res.statusCode = 200;
  res.setHeader("Content-Type", "text/html");
  app(write);
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
!END CREATE_FILE

The reason `write` is so complex is because everything in Node is asynchronous, so we need to be sure to call `end` *only* when all writes are complete.  By creating our own `write` function, we ensure that.  This complexity will
make sense later, trust me.

If we re-run our server:

!SHBG{pause=2} node js/server.js

and navigate to `http://localhost:3000`, everything is still working:

!SCREENSHOT "Dynamic Web Page" http://localhost:3000 served_page2.png

!STOPBG node js/server.js

The contents of `js/server.js` are pretty generic.  They don't have anything to do with our use-case of serving up a dynamic web
page, and we could use `js/server.js` with pretty much any function that returns a string.

Meanwhile, *our* code has no knowledge of the fact that it's run in a web server.  It's just rendering strings.  So far, this is basic separation of concerns.

But consider how you'd maintain and evolve this system.  As we just saw, we made changes to the code and had to re-run everything
to make sure it still worked.  It would be better to have an automated test.  To test our system as it stands, we'd need a unit
test of `js/app.js`, and a system test that running `node js/server.js` properly rendered teh web page.

If, instead, *we* didn't have to worry about `js/server.js`, that's a big piece of the system not to have to worry about.
Instead, we assume someone else provides `js/server.js` and, as long as our exported function conforms to a known protocol, we're
good.

## Protocols

A _protocol_ is an agreement between code that calls our code and our code.  Currently, the protocol we've established is that
our module exports a function that takes a function it can use to render text to the caller.

We can write any number of such modules, and as long as they export a no-arg function that returns an HTML string, `js/server.js`
can execute them.

To demonstrate this, let's replace `js/app.js` entirely:

!CREATE_FILE js/app.js
module.exports = (write) => {
  write((new Date()).toString());
}
!END CREATE_FILE

Now, we re-run our server:

!SHBG{pause=2} node js/server.js

If we navigate to http://localhost:3000, we see the date.

!SCREENSHOT "Dynamic Web Page" http://localhost:3000 served_page3.png

!STOPBG node js/server.js

Hopefully, you're starting to see where we're going.  If we had a programming model where it were possible to create full-fledged
applications, but where we  *only* had to write code specific to our problem domain, that would result in a lot less code with
fewer tests.  We'd be able to ship more quickly.

*But*, this still just looks like a web framework.  This doesn't seem that revolutionary, and I'd agree.  Let's expand the scope
of our application beyond serving web pages and see where that takes us.
