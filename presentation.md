
Zero to Node: Node.js in Production
===================================

* * * * *

## The Problem

* Users want to learn how to pronounce arbitrary text.

* We have a command-line text-to-speech (TTS) application.

* But our legacy PHP app to generate audio is broken.

* * * * *

## The Solution

### A node.js app.

* Thus was born **cicero**.

### What's this app really doing?

* Taking in a string and:

    * serving audio from cache, or
    * generating audio, serving it, and saving it in cache.

* Needs to be quick and stable.

* Needs to serve ~500 requests/minute.

* * * * *

## Getting started

* nodejs.org has install instructions.

* Helpful: NVM (Node version manager).

* Third-party packages via Node package manager (NPM).

* Specify dependencies in `package.json`. NPM does the rest.
<pre><code class="javascript">"dependencies": {
  "config": "0.4.15",
  "express": "3.x",
  "winston": "0.6.2"
},
"devDependencies": {
  "coffee-script": "1.3.3",
}
</code></pre>

* * * * *

## Dev Environment

* Grunt tasks: watch, test, build, server.

* Coffeescript: a must (for me).

<pre><code class="javascript">// JavaScript               # CoffeeScript

fn = function (args) {      fn = (args) ->
  return args;                args
};

var obj = {                 obj =
  key: value                  key: value
};
</code>
</pre>

* * * * *

## Express and Connect

Express (HTTP framework) and Connect (middleware) provide flexible flow control:

<pre><code class="coffeescript"># Create Express app.
express = require 'express'
app     = express()

# Set up a route.
app.get '/audio', (req, res) ->
  # Process request...
  # When finished, send response:
  res.send 200, responseData

# Add middleware.
middle = (req, res, next) ->
  # Process request...
  # When finished, can short-circuit:
  return res.send 200, responseData if allDone
  # Or can move on to next middleware:
  next()

app.use middle
</code></pre>

* * * * *

## Structure and Decomposition

Really up to you. Here's our project layout:

    /cicero
       |-config
       |-docs
       |-lib               (<--- compiled JS)
       |  |--index.js
       |  |--tts.js
       |-node_modules ...
       |-src               (<--- CoffeeScript)
          |--index.coffee
          |--tts.coffee


* * * * *

## Structure and Decomposition

`tts.coffee` exposes middleware functions: `(req, res,
next)`:

<pre><code class="coffeescript">module.exports =
  parseArgs:   TtsArgs.parse
  searchCache: TtsCache.search
  exec:        TtsExec.exec
</code></pre>

Attach middleware to routes in `index.coffee`:

<pre><code class="coffeescript">tts    = require "./tts"
middle = [log, tts.parseArgs, tts.searchCache, tts.exec]
app.get "/audio", middle
</code></pre>

* * * * *

## Async

* I started cicero doing filesystem operations with sync methods like
  `fs.openSync`.

* That's a problem: Node is single-threaded.

* Anything that blocks can halt the entire process.

* * * * *

## Callbacks

First async paradigm.

<pre><code class="coffeescript">fs.open path, 'r', (err, file) ->
  # In callback function:
  # Short-circuit and handle error if err is non-null:
  return handleError err if err

  # Otherwise, all is good, and we can do
  # something with file now that it's ready...
</code></pre>
* * * * *

## Events

Second async paradigam.

    # Create our process object.
    ttsProcess = spawn 'tts', args, options

    # Listen for exit event.
    ttsProcess.on "exit", (code) ->
      # Check errors, exit.
      if stderr or code isnt 0
        return new Error(stderr ? "TTS failed")

    # Listen for error event.
    stderr = ""
    ttsProcess.stderr.on "data", (data) ->
      # Accumulate data from stderr.
      stderr += data

* * * * *

## Streams

* Streams are a powerful way of thinking about IO.
* Send output of something directly as input to something else (like pipes in
  Unix).
* Streams can be readable, writeable, or both.
* Can pipe readable into writeable.

### Why use streams?

* Avoid buffering data in memory. Send it as available/ready from the OS.
Smooths out CPU and network load.
* Easy to reason about.

* * * * *

## Streams

Stream audio file to HTTP response (`res`) and S3 cache (`s3Stream`):

<pre><code class="coffeescript"># Get readable stream reference to audio file.
fileStream = fs.createReadStream tmpAudioFile

# res is our HTTP response object, and it's a writeable stream.
# s3Stream is our writeable stream to Amazon S3.
# Let the streaming begin!
fileStream.pipe(res)
fileStream.pipe(s3Stream)

# Listen for error events.
fileStream.on "error", errHandle
s3Stream.on "error", errHandle

# Cleanup on end event.
fileStream.on "end", ->
  # Do cleanup (delete tmpAudioFile)
</code></pre>

* * * * *

## Logging

Gain visibility into what your app is doing.

* Winston library
    * Allows multiple transports (e.g., console, filesystem, third-party service)
    * Logs JSON:
<pre><code>{
  "date": "2012-10-28T15:17:02.513Z",
  "level": "info",
  "env": "production",
  "addr": "0.0.0.0",
  "port": 2003,
  "vers": "v0.8.9",
  "message": "Server started on port 2003."
}
</code></pre>

* * * * *

## Provision and Deploy

* Many ways you could do this.
* We use Chef and Amazon Web Services (AWS).
* Find Chef cookbooks online and customize when necessary.
* Time to deploy? Update version and Chef takes of the rest.

* * * * *

## Monitoring

* Again, many ways to do this.
* We use a service called Scout.
* Scans output from logging.

<img src='deck/img/scout.png'>

* * * * *

## Summing Up

### Results

* http://audio.spanishdict.com/audio?lang=en&speed=25&text=java+script works
* Cicero meets our needs

### Node Challenges

* Thinking async.
* Newness/rapid development.
* DIY / FIOY (figure it out yourself).
* Minimalist aesthetic sometimes extends to docs.

* * * * *

## Thanks!

@williamjohnbert

http://williamjohnbert.com

SpanishDict is hiring! http://spanishdict.com/careers
