---
title: Implementing Haskell's Data.List inits and tails Functions in Clojure
category: clojure
---

[Haskell](http://haskell.org)'s
[Data.List](http://hackage.haskell.org/package/base-4.6.0.1/docs/Data-List.html)
has some interesting functions. I've been playing around with a few
recently and decided to see if there were Clojure equivalents, and if
not, what it would take to implement them.

The first function is `tails` which returns the final segments of the argument, longest first:

``` haskell
λ> tails [1,2,3,4,5]
[[1,2,3,4,5],[2,3,4,5],[3,4,5],[4,5],[5],[]]
```

Before looking at the actual implementation, I assumed (incorrectly[1])
that it would use `tail` and recurse over and accumulate the list
until it was empty. Haskell's `tail` is similar to `rest` or `next` in
Clojure:

``` haskell
λ> tail [1,2,3,4,5]
[2,3,4,5]
```

``` clojure
user> (next [1,2,3,4,5])
(2 3 4 5)
user> (rest [1,2,3,4,5])
(2 3 4 5)
```

I messed around with `loop`/`recur` for a bit but then I remembered `iterate`:

``` clojure
user> (doc iterate)
-------------------------
clojure.core/iterate
([f x])
  Returns a lazy sequence of x, (f x), (f (f x)) etc. f must be free of side-effects
user> (def s '(1 2 3 4 5))
#'user/s
user> (take 2 (iterate rest s))
((1 2 3 4 5) (2 3 4 5))
user> (take 3 (iterate rest s))
((1 2 3 4 5) (2 3 4 5) (3 4 5))
```

Great. `iterate` will just keep going because it returns a lazy
sequence, so using `take` we can limit the result to just what we
need:

``` clojure
user> (take (count s) (iterate rest s))
((1 2 3 4 5) (2 3 4 5) (3 4 5) (4 5) (5))
user> ;; ooops, Haskell's tails goes one more and gives us the empty list, []
user> (take (inc (count s)) (iterate rest s)))
((1 2 3 4 5) (2 3 4 5) (3 4 5) (4 5) (5) ())
user> ;; note that we want rest here instead of next
user> (take (inc (count s)) (iterate next s))
((1 2 3 4 5) (2 3 4 5) (3 4 5) (4 5) (5) nil) 
user> ;; ok, lets wrap it up into a function and test it out on something else ...
user> (defn tails [xs] (take (inc (count xs)) (iterate rest xs)))
#'user/tails
user> (tails ["what" "is" "for" "lunch"])
(["what" "is" "for" "lunch"] ("is" "for" "lunch") ("for" "lunch") ("lunch") ())
user>
```

Hrm, it still works but now we have a mixture of types in the
result. This may or may not bother you depending on what you plan to
do with the result. We could normalize it using `map` and `sequence`:

``` clojure
user> (defn tails [xs] (map sequence (take (inc (count xs)) (iterate rest xs))))
#'user/tails
user>  (= (tails '(1 2 3 4 5)) (tails [1 2 3 4 5]))
true
user> 
```

Or, with a little more work we can make the resulting collections the same type as the one passed in:

``` clojure
(defn tails [xs]
  (let [f (cond
           (vector? xs) vec
           (set? xs) set
           :else sequence)]
    (map f (take (inc (count xs)) (iterate rest xs)))))
```


``` clojure
user> (tails [1 2 3 4 5])
([1 2 3 4 5] [2 3 4 5] [3 4 5] [4 5] [5] [])
user> (tails '(1 2 3 4 5))
((1 2 3 4 5) (2 3 4 5) (3 4 5) (4 5) (5) ())
user> (tails #{1 2 3 4 5})
(#{1 2 3 4 5} #{2 3 4 5} #{3 4 5} #{4 5} #{5} #{})
user> 
```

Next up is a similar function, `inits`, which is sort of the inverse of `tails`:

``` haskell
λ> inits [1, 2, 3, 4, 5]
[[],[1],[1,2],[1,2,3],[1,2,3,4],[1,2,3,4,5]]
```

`inits` starts with an empty list, and then slowly builds upon the
argument until the final item is the full list given. Here was my first attempt:

``` clojure
user> (def xs [1 2 3 4 5])
#'user/xs
user>  (reverse (take (inc (count xs)) (iterate drop-last xs)))
(() (1) (1 2) (1 2 3) (1 2 3 4) [1 2 3 4 5])
user> 
```

Aside from the reverse, we basically switch out rest for
drop-last. This works, but the reverse kind of feels like cheating to
me. Here is a less hacky option: a list comprehension. We'll package
it up into a function as well to hide the details:

``` clojure
(defn inits [xs]
  (for [n (range (count xs))]
    (take n xs)))
```

``` clojure
user> (inits xs)
(() (1) (1 2) (1 2 3) (1 2 3 4))
user>
```
And as above, retaining the collection type still applies.

There are a bunch of other good ones in `Data.List`. I might try to talk about more later.

