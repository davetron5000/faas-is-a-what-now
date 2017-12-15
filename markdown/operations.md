At this point, we have a system that exemplifies a functions-only, event-sourced architecture.  We can see the advantages in separating plumbing (like routing web requests) from our business logic.  If we assume that our cloud
service provider (like AWS) handles that plumbing and event-routing for us, that means that the thing we deploy to production is a bunch of functions that adopt a protocol.  Our cloud provider will be in charge of making them execute.

We've seen some benefits from a system design perspective, but this also has great _operational_ benefits.

In a typical MVC-style web app, the unit of deploy is an entire application and we're expected to manage all parts of it, including what the framework provides.  Below is the architecture of such a system, with `«framework»` denoting parts of the framework we didn't write, but are running, and `«managed»` denoting components our cloud provider handles.  Anything not `«managed»` is our responsibility, even if we didn't write it:

!GRAPHVIZ mvc_app.png "Architecture of an MVC-style web app:
digraph faas {

  nodesep=0.5
  rankdir="TB"

  node[fontname="Avenir" fontsize="16" margin="0.4,0.3"]
  edge[fontname="Avenir" fontsize="16"]

  BackgroundJob   [ label="Background Job subsystem" shape="component"]
  Controller      [ label="Controllers" shape="component" ]
  Database        [ label = "RDBMS\n«managed»" shape="box3d"]
  DatabaseAdapter [ label="RDBMS adapter\n«framework»" shape="signature" ]
  EmailProvider   [ label="Email Transport\n«managed»" shape="box3d"]
  Mailers         [ label="Mail subsystem\n«framework»" shape="signature"]
  Model           [ label="Models" shape="component"]
  QueueingSystem  [ label="Job Queue\n«managed»" shape="box3d"]
  Router          [ label="Internal request routing\n«framework»" shape="signature"]
  ServiceObject   [ label="Business Logic" shape="component"]
  WebRequest      [ label = "Web requests\n«managed»" shape="rarrow"]

  WebRequest    -> Router
  Router        -> Controller
  Controller    -> Model
  Controller    -> ServiceObject
  ServiceObject -> Model
  ServiceObject -> Mailers
  ServiceObject -> BackgroundJob
  BackgroundJob -> QueueingSystem
  Mailers       -> EmailProvider
  Model         -> DatabaseAdapter
  DatabaseAdapter -> Database

}
!END GRAPHVIZ

<aside class="pullquote">We are responsible for the operations of that application <em>including the framework and library code</em></aside>

I've been pedantic about framework-provided subsystems for a reason.  Because we are asking our hosting provider
to run our entire application, we are responsible for the operations of that application _including the framework
and library code_.  Our hosting provider doesn't care that some open source team maintains our database adapter—we
are the ones choosing to run it in production.

If we compare that to our functions-as-unit-of-deploy, there's less stuff to manage, and each function has a
smaller footprint.

Consider if our tiny application was implemented as an MVC-style web framework.  We would need to have visibility into the
behavior of every piece of the framework as described in the above diagram. To understand how our application performs, we have
to examine both our code and the framework code.  If you've used an Application Performance Monitoring (APM) tool like New Relic, you know how complex this can be, especially if your web framework or programming language doesn't have a lot of hooks for it
to instrument.

You end up having to litter your code with stuff like this:

```javascript
db.createObject(
  // "nr" is the custom tracing library
  nr.createTracer('db:createObject', function (err, result) {
    if (util.handleError(err, res)) {
      return
    }
    res.write(JSON.stringify(result.rows[0].id))
    res.write('\n')
    res.end()
  })
)
```

When we design our system as functions responding to events, all of which is managed by the cloud services provider, it gets
simpler and we don't need very much custom tracing junk:

!GRAPHVIZ function_architecture.png "Our system as functions"
digraph faas {

  nodesep=0.5
  rankdir="TB"

  node[fontname="Avenir" fontsize="16" margin="0.4,0.3"]
  edge[fontname="Avenir" fontsize="16"]

  subgraph cluster_saveEmail {
    fontname="Courier"
    label="ƛ saveEmail()"
    SaveEmail
    DatabaseAdapter
  }
  subgraph cluster_sendEmail {
    fontname="Courier"
    label="ƛ sendEmail()"
    SendEmail
    Mailer
  }
  subgraph cluster_renderHTML {
    fontname="Courier"
    label="ƛ renderHTML()"
    RenderHTML
  }
  Database        [ label="RDBMS\n«managed»"           shape="box3d"]
  DatabaseEvent   [ label="Database event\n«managed»"  shape="rarrow"]
  DatabaseAdapter [ label="RDBMS adapter\n«library»"   shape="signature" ]
  EmailProvider   [ label="Email Transport\n«managed»" shape="box3d"]
  SaveEmail       [ label="Save Email"                 shape="component"]
  Mailer          [ label="Mail subsystem\n«library»"  shape="signature"]
  SendEmail       [ label="Send Email"                 shape="component"]
  RenderHTML      [ label="Render HTML"                shape="component"]
  WebRequest      [ label = "Web requests\n«managed»"  shape="rarrow"]

  WebRequest      -> SaveEmail
  WebRequest      -> RenderHTML
  SaveEmail       -> DatabaseAdapter
  DatabaseAdapter -> Database
  Database        -> DatabaseEvent
  DatabaseEvent   -> SendEmail
  SendEmail       -> Mailer
  Mailer          -> EmailProvider
}
!END GRAPHVIZ

We're still using the same general components, but a few things are different:

* Monitoring our system becomes monitoring each function independently.  We don't need to add custom tracing in
our code because it's naturally separated as deployable functions, and our cloud hosting provider can provide
APM-style insights.
* Each “thing we are monitoring” is simpler - instead of monitoring one large application with many subsystems, we
are now managing three smaller functions with few subsystem each.  For example, we don't have to worry about email
or request routing when examining the `saveEmail` function.
* Because we don't have one integrated system, we no longer need to use a framework.  We can use the best RDBMS
adapter, the best email subsystem, the best HTML rendering engine for the problem at hand.  While an integrated
web application benefits greatly from an integrated framework, our “free functions” don't, so we can use different
libraries if it's warranted.
* To change any given function, we only have to deploy that function.  We don't need to run a test suite of the
entire system just to deploy a change to one component.  As long as our functions conform to the protocol, we are
good (this is not without downsides, which we'll get to in a later chapter).

There are theoretical cost benefits, too.  Our function that renders HTML likely gets called a lot, but our function that sends
email likely not nearly as much.  The way most Faas providers work, you only pay for when your function executes, as opposed to
paying for server processes or virtual machines that are always on.

So, a serverless architecture based on event-sourcing, hosted by a cloud Faas provider has many advantages that we've seen.  The
programing model is attractive, the operational model is simplified, and it could cost less.  There can't be any downsides, can
there?

