---
title: Haskell: http-streams and Aeson
category: haskell
---

### Learning Haskell

I've been learning [Haskell](http://haskell.org) off and on for the
past few months. It's pretty awesome, but definitely a challenging
language to learn. I've had exposure and experience with functional
programming for a couple years now (first with Scala, then/currently
with Clojure, and some Erlang), and while those concepts are fairly
well cemented in my brain, Haskell has a bunch of other stuff that
make it a challenge. That's not to say this is a bad thing; in my
limited experience, I've been finding that some of what initially
seems complex turns more towards elegance once I'm comfortable with
it, and Haskell rewards your efforts in many areas by being a really
nice way to get things done. I figured I'd write up something here
based on a recent challenge I faced and how with a bit of Googling and
playing around I managed to figure it out and move on.

I'll usually try to bite off a small project in a new language once I
feel reasonably comfortable with the basics. In the past I've found
that there is one type of small project that makes a decent first try:
RESTful API clients. There are a few reasons why:

* Usually only requires a couple of external dependencies, if any (HTTP and JSON/XML parsing)
* Easy to incrementally build and test (i.e. only implement a single API call initially, then go back and fill in)
* Lots of APIs and variation to choose from
* Possibly useful to you and/or someone else out there

For a simple walkthrough, I've chosen the USGS Geojson feeds that
contain recent earthquakes broken down by recency and size. It's more
of a "feed" than an API, but it works well because it requires no
signup or authentication, and just lets you start pulling JSON off the
web to see what happens. The list of feeds can be found here:
http://earthquake.usgs.gov/earthquakes/feed/v1.0/geojson.php

### Dealing with HTTP

There are a few options for HTTP client libraries, but I've chosen
[http-streams](http://hackage.haskell.org/packages/archive/http-streams/0.6.0.1/doc/html/Network-Http-Client.html). 
It may or may not be the best/right choice for this application ... but I
wanted to see how it worked. Getting something running with the basics is super easy. http-streams has some convenient wrappers for basic get requests:

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=HTTPStreamsSimple.hs"> </script>

We are all pretty used to simplified HTTP stuff these days, and
Haskell is no exception here. We hand <code>get</code> a full URL and
some handler function that will be called with the
<code>Response</code> and an <code>InputStream</code>. To make things
even easier when we want to see what the body looks like, the built in
<code>debugHandler</code> will dump out the response header and body
to stdout for us to peruse. If we just wanted the body alone on
standard out, we can replace the above call to
<code>debugHandler</code> with <code>(\p i -> Streams.connect i stdout)</code>.This just connects the body stream to standard out and
ignores the response headers.

If you hate trying to read unformatted JSON, you can install
[aeson-pretty](http://hackage.haskell.org/package/aeson-pretty) which
has a command line tool to pretty-print JSON:

<code> HTTPStreamSimple | aeson-pretty </code>

Or with python: 

<code> HTTPStreamSimple | python -m json.tool </code>

If we want to start thinking about building a library around this,
though, we need to change a couple of things. First, we probably want
a little more control over the request so we can abstract it away from
just a full URL every time, and second, we need to get the body into
some intermediate state so we can parse it and return something more
useful. This next snippet does just that, with the same result as our code above.

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=HTTPStreams.hs"> </script>

Now we are openning a connection to the host on port 80, building a
request that contains the request type, the path, and saying what we
expect in the response with <code>setAccept
"application/json"</code>. If we were sending a POST request or needed
to send some other arbitrary header information, now would be the
right time for that as well. Next, we send the request with an emtpy
body (since it's a GET). We can now receive the response, and things
start to get familiar again as the <code>receiveResponse</code> call's
second argument is the same type as we used above with
<code>get</code>. The difference, however, is that we are using the
built in <code>concatHandler</code> which will give us access to the
body rather than just dump it out to the terminal. For now
that is all we are doing with <code>S.putStr x</code>. Finally we close the connection.

Later on we might handle the
response and keep that code clear of the actual protocol
layer. In a real project we might have a few layers of
abstraction, but for this post we'll keep it fairly simple and just
take advantage of having an easy point of entry with the handler. Now
we need to hook up some parsing action with
[Aeson](http://hackage.haskell.org/package/aeson). This is actually
the point at which I had some trouble, and thus the inspiration for
this post, but before I explain that, here is a quick look at using
Aeson so we can see what we are dealing with API-wise.

### Parsing JSON

Looking at the Aeson project's
[examples](https://github.com/bos/aeson/tree/master/examples), you'll
see that you really have four options for dealing with JSON data
to/from Haskell: the standard approach, a [Template
Haskell](http://www.haskell.org/haskellwiki/Template_Haskell) option,
and two generic options. I'm too green to suggest the "right" one, but
so far I'm a fan of using the one that requires the least amount of
code up front, and then slowly migrating to the one that provides the
clearest implementation in the end, so we'll start off making it easy
on ourselves and use one of the generic methods and then ditch it for
the standard approach which will require more work on our part but
will also allow us to have more control of our types. Let's take a
look at something similar to those examples but modified for our
earthquakes. For now I'll just put in a few fields of our main
types to keep it simple. If Aeson sees stuff that isn't in our type
definition it'll just skip it (though if we have it and it's missing
we'll get a parse error, more on that later). We'll also leave out the
ToJSON stuff since we don't have a need to convert back to JSON in
this case:

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=AesonSimple.hs"> </script>

This is pretty basic. We define four main types: <code>MetaData</code>, <code>Properties</code>,
<code>Feature</code> and <code>Feed</code>. Working from the bottom up, Feed is our top level
container. Notice that it consists of a chunk of metadata and then a
list of (zero or more) Features with <code>[Feature]</code>. A Feature
itself contains a properties slot and (for now) just the Feature
ID. Similarly we've left off most of the Properties items and just
added the detail String and the mag (magnitude) Double. Finall we have
a few slots in our MetaData type.

If we keep heading up to the top of the file, we see that we've
imported the GHC.Generics which lets us get away with not having to
tell Aeson how to parse our JSON (coupled with the DeriveGeneric
extension at the top of the file), and we're also importing
Data.ByteString.Lazy.Char8 because Aeson actually works on ByteStrings
but we get away with using String  by also including the OverloadedStrings
extension. 

To complete our tour of this file, jump back down to main. We call
decode on some inline JSON and tell it we want a Maybe Feed. This is
handy: if parsing fails for any reason, we'll get back
<code>Nothing</code>, otherwise we should have <code>JustFeed</code>. 
Using a case statement we can pull out our result and
print it out. We derived <code>Show</code> in all of our types so we
get a nice representation of the type on the terminal for inspection.

This is now the point at which I got a bit stuck. In theory we have all the
parts we should need to fetch the JSON and parse it into our data
structure(s), but there was a problem: Aeson expects a lazy ByteString
to decode, but what we are getting from http-streams (or from the
underlying io-streams, I guess) is strict. Here is what I did:

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=HTTPStreamsAeson.hs"> </script>

In jsonHandler, I'm using <code>Streams.toList</code> which should
give me the whole body as a list of chunks. This ensures that we get
all the parts from a large body so we can correctly parse the
JSON. [fromChunks](http://hackage.haskell.org/packages/archive/bytestring/0.9.1.5/doc/html/Data-ByteString-Lazy.html#v:fromChunks)
let's us take a list of strict <code>ByteStrings</code> and converts
it to a single lazy ByteString, which we can hand directly to Aeson's
<code>decode</code> to parse our JSON. Rad.

I'm hoping this is the right way to go. After searching a bit, I
found out that I may not be the only one with this issue, and in fact,
there is an [issue](https://github.com/bos/aeson/issues/99) logged
against Aeson with a workaround. This would still require that the
body be converted to a single strict ByteString, however, so I'm not
sure if it gains much in the way of convenience or performance.

For the final revision, I've done a few things. First, as mentioned
earlier, there are a few options with Aeson. I started out with the
simple Generics method that let us write the least amount of code to
start parsing some JSON into our types. From what I can tell, though,
this backs us into a corner with our type's field names; if they don't
match the JSON keys, we don't get them. The other problem is that we
don't have the ability to transform types, for instance, we might want
to parse a date into a specific date type rather than just using a
String. Third, if you happen to be grabbing JSON with optional fields,
going to the standard approach will let you handle these
correctly. Basically you have the typical tradeoffs: more code to
write, debug and maintain in exchange for more flexibility and tighter
types.

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=HTTPStreamsAeson2.hs"> </script>

The next things to notice: I pulled the HTTP call out of main and
wrapped it nicely in its own function, and added some parameters to
build the URL. Now we have a more DSL feel, passing in our required
magnitude and timeframe. The result is still a list of feeds, and
after making sure we have some we do some contrived stats on the
magnitudes.

Haskell is pretty nice. Feedback welcome from newbies and seasoned Haskellers alike.


(Edit: after posting this I found a similar example [here](https://gist.github.com/fujimura/5388019) by [Fujimura Daisuke](http://fujimuradaisuke.com/))
