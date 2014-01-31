---
title: Functions in Data.Maybe
category: haskell
---

Most introductions to monads in Haskell talk about `Maybe`. Here are
some of the handy functions in that module for operating on the type.
`maybe` (the first function below) is should be available in ghci
right away in Prelude, but the others require that you load the
Data.Maybe module in ghci:

``` haskell
λ> :m +Data.Maybe
```

`maybe` either applies a function to a `Maybe` or returns a default
value. It's type signature looks like this:

``` haskell
maybe :: b -> (a -> b) -> Maybe a -> b

λ> let x = Just 40
λ> let y = Nothing
λ> maybe 30 (*2) x -- x is Just 40, so it gets applied to (*2)
80
λ> maybe 30 (*2) y -- y is Nothing, we end up with the default value
30
```

`isJust` is straightforward. Given a `Maybe` argument, it returns true if it's `Just`:

``` haskell
isJust :: Maybe a -> Bool

λ> isJust x
True
λ> isJust y
False
```

`isNothing`: not hard to figure what this does:

``` haskell
isNothing :: Maybe a -> Bool

λ> isNothing x
False
λ> isNothing y
True
```

`fromJust` is a relatively unsafe way to get at the value of a `Maybe`:

``` haskell
fromJust :: Maybe a -> a

λ> fromJust x
40
λ> fromJust y
*** Exception: Maybe.fromJust: Nothing -- woops. use a case expression in real code
```

`fromMaybe` is similar to `maybe`, but instead of a function argument
being applied to the value, you simply get the value in your `Maybe`
or the default you pass in:

``` haskell
fromMaybe :: a -> Maybe a -> a

λ> fromMaybe 30 x
40
λ> fromMaybe 30 y
30
```

`maybeToList` will plop your `Maybe`'s value into a list and return it:

``` haskell
maybeToList :: Maybe a -> [a]

λ> maybeToList x
[40]
λ> maybeToList y
[] -- always get a value, it just happens to be an empty list if we pass in Nothing
```

`listToMaybe` goes the other way:

``` haskell
listToMaybe :: [a] -> Maybe a

λ> listToMaybe [2]
Just 2
λ> listToMaybe $ maybeToList x
Just 40
λ> listToMaybe $ maybeToList y
Nothing
λ> listToMaybe [20,40,60] -- listToMaybe will drop all but the first element
Just 20
λ> listToMaybe "foo"
Just 'f'
```

`catMaybes` takes a list of `Maybe`s and returns a list of the values
for any `Just` value, but drops out your `Nothing`s:

``` haskell
catMaybes :: [Maybe a] -> [a]

λ> catMaybes [x, y, (Just 21), Nothing, Just(22)]
[40,21,22]
```

`mapMaybe` lets you apply a function that accepts and returns a
`Maybe` to a list of `Maybe`s, but returns a list of the values (and
skips the `Nothing`s):

``` haskell
mapMaybe :: (a -> Maybe b) -> [a] -> [b]

λ> let f x = (+2) <$> x -- or fmap (+2) x
λ> f (Just 30)
Just 32
λ> mapMaybe f [(Just 20), Nothing, x]
[22,42]
```
