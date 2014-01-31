---
title: Erlang function options
category: erlang
---

In working on a small Erlang project I found myself writing a few
functions that take one or more required parameters and then possibly
a few optional ones. A common idiom in Erlang is take a
list of tuples and/or atoms for optional parameters. For example,
`gen_tcp:connect` has a signature like this: `gen_tcp:connect(Host, Port, Options)`, where Options can be all kinds of stuff, such as:

``` erlang
{ok, Socket} = gen_tcp:connect("localhost", 12345, [binary, {packet, 0}, {active, false}]).
```


In this case, the host and port are required, and the third is a list
 of various kinds of meta data, in various forms. `binary` is a single
 atom; `packet` takes an integer, and according to the
 [docs](http://erldocs.com/R16B02/kernel/inet.html#setopts/2) (which
 are actually part of inet:setopts) the value can be either 0 or `raw`, or
 `1`, `2` or `4`. The value of `active` can be `true`, `false` or `once`.

I'm using a function to build up a record that will
then be passed on to another function for further processing. The user
should only have to pass in one required parameter, and then a list of
zero or more optional ones, so I want the call would look something like this:

``` erlang
{ok, Result} = my_func(Arg2, Arg2, Options).
```

where `Options` looks like `[foo, {bar, 20}, {baz, "hello"}]`.

In this case no options are required, so I also added in a 1 parameter version of the function:

<script src="https://gist.github.com/joshrotenberg/7666492.js?file=function_options.erl"> </script>

Pattern matching and recursion make handling this kind of thing pretty
clean. If I need to add an option or change how one is handled, its
clear where to do it and its easy to test those changes.

