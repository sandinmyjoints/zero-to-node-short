## Slide 1 - Title

* Hi, thanks for coming.
* I'm William Bert, a software engineer at SpanishDict.
* My talk is titled Zero to Node. It's a case study of deploying node.js in
  production based on a project I worked on in August, my first month here.
* This is the lightning version.

## Slide 2 - Problem

* SpanishDict.com is an English-Spanish language learning website. Part of
teaching language is teaching pronunciation. Pronunciation is taught by hearing
words pronounced correctly.
* Users are arbitrary--they want to hear how to pronounce whatever words they
  want to learn!
* We have a text to speech (TTS) product that actually does a pretty good job of
pronouncing arbitrary Spanish and English text (play Spanish sample).
* The TTS application is enterprise software that only runs on the command line,
but our old PHP app is not working well and not worth repairing.

## Slide 3 - The Solution/Yikes/What's it really doing

* The solution is (you may have guessed) a node app.
* Thus is born Cicero, my first foray into node.

* What's this app really doing?
* Taking in a string and

    * serving audio from cache, or
    * generating audio, serving to user, and caching

* In other words, **slinging data**. Node can do that really well.
* We need to serve the audio quickly (very quickly) and efficiently.
* This isn't a core service, but a nice add-on.
* We just want to forget about it. We want stability, relatively low resources,
simplicity so easy to maintain.
* We'll serve ~500 requests/minute.
* So now the problem is well understood.

## Slide 4 - Getting started/install/dependencies

* Install is easy on any platform. Here's how to do it on OS X with Homebrew
  (see slide).
* We installed `nvm`, node version manager, which allows you to have
  multiple node versions installed at once. This is useful because node is
  updated all the time.
* This way, you can try out new versions and switch when ready. Different apps
  can use different versions.

* Node has **lots** of third-party packages.
* We used Node package manager (npm) to install some packages out of the gate
  to get the most out of node. We'll talk about what they do in a minute.
* Put a list of packages and versions in `package.json` (see docs for
  boilerplate).
* Do `npm install`.
* Npm installs everything and their dependencies into `node_modules`.

## Slide 5 - Dev Env

* I'll briefly describe my dev environment.
* I use a command line dev tool called grunt. Very useful for dev tasks that
  come up over and over: watch, test, server. Kind of like Rake in Ruby.
* Coffescript is "a little language that compiles to Javascript." You write
  Coffee-script but you run JS.
* Coffescript is really "the best parts of Javascript". Protects you from
  tripping on some gotchas like global variables, semicolon insertion, weird
  type coercion.
* It passes JSLint static analysis without warnings, works in all JS runtimes,
* Coffescript also looks better than straight JS: less verbose, more readable,
  uses significant whitespace. It reminds me of Python.
* Quick CS explanation (see slide)
* My examples are in CS. Ask if anything is unclear about it.

## Slide 6 - Express and Connect

* Express is a node HTTP application framework built on Connect, which is a
  middleware framework.
* They're so much part of my node experience that I almost forget they are their
  own things because they provide essentials: an HTTP application, and its
  routes and middleware. (see slide)
* Routes are where requests to a url go. Typically they end in sending
  something back to the user via response.
* Middleware allows common functionality to be shared among routes. So we can
  pass all our requests through a chain of middleware.
* Middleware are just functions that take req, res, do something, and call the
  next middleware function.
* Example: logging. Any request we get to any route, we want to log it, so we
  pass to log middleware first, it logs the request, then passes to next.
* Now, sometimes you might want to short-circuit in middleware and send something back to
  the user right then and there. You can do that.
* Exmaple: if there was a request to '/audio' for some text, and in the cache-lookup
  middleware we find the audio for that text, then we can just send that audio
  data without going any further.
* We can send some routes through some middleware, like auth, but others will go
  through different middleware.
* Seemingly simple, extremely flexible.

## Slide 7 - Structure and Decomposition - Project layout

* Node is ripe for structuring. It's entirely up to you, unlike Django or RoR.
* Here's Cicero's directory structure.
* We have this src directory at the bottom that holds Coffee-script.
* index.coffee and tts.coffee are the main source code files.
* index.coffee has app config, logger setup, top level routes.
* tts.coffee has app logic: turn params into args, exec, cache, respons
* grunt has a build task that compiles from src/server into lib. Coffee goes in,
  Javascript comes out. `lib` directory is a standard place for app code.

## Slide 8 - S and D - Connect app routes with app logic via middleware

* So we're going to connect our app skeleton in index with our app logic in
  tts.coffee using middleware.
* TTS exposes several middleware functions (which are just functions that take `(req, res,
  next)` and do something, then call next or return a response).
* parseArgs, searchCache, exec.
* Then we attach logging middleware and our TTS middleware to the /audio route.
* So every request goes through this chain:
  * log request
  * process args
  * search cache, return if found
  * exec tts command line application if not in cache, serve to user and store
    in cache

## Slide 9 - Async

* So now that we've laid out the app structure, I'm going to discuss one of the
  techniques used within the app: async control flow.
* I looked at some early commits and found sync methods.
* Sync is a problem because Node is single-threaded, so one blocking call can
halt everything in the entire Node process, or if not halt, then slow down.
* Node is designed for async, everything in the std lib and the ecosystem
  supports async, and async works well in practice for web apps, so I wanted to
  be sure to use it.
* Async: I understood it in theory, but had limited exposure in practice.

## Slide 10 - Callbacks

* First async paradigm: callback functions.
* They are functions that are called once something is ready. In other words,
  they are deferred execution.
* They are kind of the building blocks of Node async.
* This example shows how to open a file.
* The flow is fairly straightfoward: start this operation, keep going with
  whatever comes next, but when this operation is done, execute this code.

## Slide 11 - Events

* Events are the next async paradigm I encountered.
* Events are like arbitrary callbacks--an event emitter object can emit them
  whenever some condition occurs, like receiving data, or encountering an error.
* (see slide) We create the process object, init stderr, and register the event
  handlers, all synchronously within one 'tick'.
* A 'tick' is one trip around the event loop. Node is continuously running this
  event loop, and on each trip around event loop: node asks, is it time to fire
  this callback? execute this event handler? etc.
* So on subsequent ticks, if our process has triggered 'exit' or 'data' events,
  the cbs execute (synchronously inside each)
* Using this event loop/async event paradigm mean node effectively never blocks;
  it only executes code when the condition that code is supposed to handle has
  occurred. This is especially useful for I/O bound applications like HTTP
  servers: we don't block for network input or filesystem operations while
  complete; they can run and node is free to keep serving requests.

## Slide 12 - Streams

* So I got comfortable with callbacks first. And events. Then Ryan says to
  switch to streams.
* Streams are another async paradigm. Powerful abstraction for IO.
* Streams are like pipes in unix. Send output of something directly as input to
  something else.
* Can be readable, writeable, both. Can pipe readable into writeable.
* Why use streams?
* Avoid buffering data in memory. Send it as available/ready from the
OS. Smooths out CPU and network load. Users see faster response.
* Easy to reason about.

## Slide 13 - Stream example

* Here's an example use of streams to return audio data (see slide).
* Note that one readable stream is piping to two totally separate writeable
  streams simultaneously. Node and the OS handle all that for us.
* Listen for events. Handle end and error events, particularly.
* Ideal would be to stream directly from the TTS application without touching the
filesystem, but the application is picky and uncooperative.

<br><br>

## Slide 14 - Logging

* Now that we have our app working, let's start thinking about operations and
  maintenance.
* Logging allows you visibility into what your app is doing.
* This is very important for production deployment so you know the app is
  working correctly, or if it breaks, you know what is going wrong.
* We use a library called Winston for logging.
* Configure multiple transports:
* log to console in dev.
* log to filesystem and a service (e.g., Loggly) in production.
* It outputs JSON instead of a big string blob.
* We can data mine these logs later for diagnosing any problems, or also for
  business value, say, what kinds of words or phrases do people tend to want to
  pronounce?

## Slide 15 - Provisioning and Deployment

* There are many ways you could do this. It's a big topic.
* We use Chef for provisioning and deployment. Chef is "an open-source systems
  integration framework built specifically for automating the cloud."
* That means we can control our servers through code.
* We write cookbooks that describe how to configure machines, then Chef executes
  those cookbooks and actually configures machines.
* AWS is our IAAS provider. They give us raw machines to use. Chef is how we
  control them.
* A lot of kinds of Chef cookbooks are available online. You can use the off the
  shelf or customize them fairly easily.
* Ryan wrote our general node webapp cookbook.
* I wrote a tts webapp cookbook which extends that for cicero.
* Once the cookbook is written, we can use Chef to start, stop, and deploy our
  app with one or two commands. Quick and convenient.

## Slide 16 - Monitoring

* In production, you can't read the voluminous logs, so monitoring aggregates
  the logs and gives even better visibility into your app's operation.
* Again, there are many ways to do this.
* We use a service called Scout.
* It simple scans output from logging, and produces realtime charts like this.
* We configure alerts so that if the number of requests drops below a threshold,
  or the amount of memory used goes above a threshold, we get emails or texts.

## Slide 17 - Summing up

### Results

* http://audio.spanishdict.com/audio?lang=en&speed=25&text=java+script works
* Cicero meets our needs
* Serves audio quickly and efficiently, without fuss

### Challenges

* Async means wrapping your head around a new kind of flow control if you're
  used to sync. Debug is a little messier, but tolerable.
* Newness and rapid development mean best practices aren't always well known,
  and change frequently. Need to keep up with the community.
* Seems to be a DIY/FIOY attitude, which you may like or not, depending on your
  temperment, your task, and your deadlines.
* Minimalism of node/express sometimes extends to docs. Aesthetically pleasing,
  but frustrating when you're trying to understand something. Always good to
  look at source.

## Slide 18 - Thanks and Questions

* So hopefully this presentation has explained, one way at least, to go from
  zero to node.
* Thanks for listening!
* I'll put a link to the slides online at my website soon.
* Quick note: If this sounded interesting, SpanishDict is hiring. Talk to me,
  Chris, or Ryan.
* Questions?
