---
title: Follow Up: http-streams and Aeson
category: haskell
---

### Followup

In my [last post](/haskell/2013/05/28/http-streams-and-aeson/) there
were (at least) a few things that could be done better. I got some
great feedback from the smart people on [Hacker
News](https://news.ycombinator.com/item?id=5801438) so I've made a couple changes to the final source.

The biggest issue is that I was completely [losing the benefit of
streaming](https://news.ycombinator.com/item?id=5803731) by reading
the entire response body in and then parsing it. I knew this but as I
mentioned I hadn't yet figured out how to handle it
correctly. Fortunately someone else does, and with their suggestions
(and a bit more Googling) I figured it out. It could probably still be
cleaned up a little but for these purposes it does the trick.

<script src="https://gist.github.com/joshrotenberg/5666409.js?file=HTTPStreamsAeson3.hs"> </script>

On line 90 there is a new function called
`parseJSONFromStream`. Calling this on the stream from
`receiveResponse` (and adding the type hint for a
`Result` wrapped around a `Feed`) gives us a
similar situation as we had with `Maybe Feed`, so the main
function does essentially the same thing still. With a little more time
we could probably make this cleaner in `fetchQuakes` by
giving the handler more to rather than calling it in a lambda.


And speaking of `fetchQuakes`, using http-streams'
`withConnection` cleans it up a little bit by
automatically handling closing of the connection. It saves a couple
lines of code and makes the function less cluttered.

Thanks again for all of the feedback!
