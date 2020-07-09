---
layout: page
title: Parallel "any" and "all"
categories: [blog, dev]
tags: [haskell, parallelizm, benchmarks]
---

# Parallel "any" and "all"

The goal of this blogpost is to demonstrate how to write in Haskell a parallelizable
computation with early short circuiting.

## Motivation

A few days ago I stumbled upon this StackOverflow question: [Parallel "any" or "all" in
Haskell](https://stackoverflow.com/questions/60160055/parallel-any-or-all-in-haskell) and I
immediately got excited because I had just the right toolset and exact solution in
mind. Moreover, I had an urge to add this functionality to
[`massiv`](https://github.com/lehins/massiv) for quite some time now. So, I thought to
myself, that I'll quickly write up the solution as an answer to the SO question and make my
array processing library just a little bit faster. Two birds with one stone, so to
speak. Turns out, not so fast, pun intended.

## Intro to the problem

For those of you that don't know, whenever you apply function `all` or `any` to a list
they will terminate as soon as a condition they are looking for is met. For example here
we know that it will terminate as soon as it reaches value `501`:

```haskell
λ> any (> 500) [1 :: Int ..]
True
```

Lazyness allows us to construct infinite lists, but short circuiting works just
as well in languages that are strict, becuase of the way `||` and `&&` operators work.

The question is, can we split a list into some chunks and let separate threads look for
this terminating condition? More importantly, can we reliably make sure that once solution
is found by one thread it will let other threads know that they can stop what they are
doing, since reagardless wht they find it will not affect the outcome?

Techincally, the last statement is not entirely true in presence of lazyness and here is
an example where sequentially we will get an answer just fine:

```haskell
λ> any even [1 :: Int, 2, 3, undefined]
True
```

While in a presence of another thread, it would be absolutely possible that the answer
will be bottom, or more specifically in the case above `*** Exception:
Prelude.undefined`. But lets ignore that fact for now.

## Attacking the problem

Lists are inherently sequential, therefore the only possible thing we would be able to do
is to iterate the list from head to tail sequentially and evaluate its elements to Normal
Form (NF) in parallel. If the question had to deal with lists of lists, then parallelizing
computation of inner lists would be possible, but even then, I know for a fact that you
can't get an optimal solution. Simply because parallel computation over linked lists is a
terrible idea. Unless computation of each element outweights the list overhead then let's
just abandon this idea in favor of something more efficient, like an unboxed array of
primitive values for example.

There are a few functions with similar properties as `any` and `all`. Direct deriviatives
`or` and `and` functions are the first ones that come to mind. Equality `(==)` and
`compare` are the other ones. Checking for presence of an element in an array or searching
through trees with something like
[BFS](https://en.wikipedia.org/wiki/Breadth-first_search) or
[DFS](https://en.wikipedia.org/wiki/Depth-first_search) are good examples as well. Because
of this similarity I will focus on function `any`, so just keep in mind that same concepts
can be applied to most of such problems.

### Establish the base case

Let's take a look at our first benchmark of searching for an even number in a long
sequence of odd numbers with a single even number towards the end. Such a scenario
simulates a good setup where a parallel algorithm with short-circuiting would do best.

This binding creates an array with 8 million odd numbers and number `2`, which is located
at around 7 million:

```haskell
arrayLate :: Array P Ix1 Int
arrayLate =
  computeAs P $
  concat' 1 [A.enumFromStepN Seq 1 2 7000000, A.singleton 2, A.enumFromStepN Seq 1 2 999999]
```

This primitive single dimensional array (denoted at type level by `P` and index `Ix1` respectfully) of
integers (`Int`) will look something like this:

```haskell
[1, 3, 5, 7, ..., 13999997, 13999999, 2, 1, 3, 5, ..., 1999993, 1999995, 1999997]
```

The way you'd check for even numbers with `any` in `massiv` is just as other data structure
or library:

```haskell
λ> import Data.Massiv.Array as A
λ> A.any even arrayLate
True
```

Upcoming is the comparison of how fast a flat primitive
[`Array`](https://www.stackage.org/haddock/nightly-2020-02-08/massiv-0.4.5.0/Data-Massiv-Core-List.html#t:Array)
from `massiv`, a primitive
[`Vector`](https://www.stackage.org/haddock/nightly-2020-02-08/vector-0.12.1.2/Data-Vector-Primitive.html#t:Vector)
from `vector` package and a list from `base` can find this number `2`.


<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2020-02-12-parallel-any-all/late-no-termination.html" style="overflow: hidden; width: 100%; height: 112px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
stack bench --ba '--output late-no-termination.html --match pattern any/Late --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv late-no-termination.html ../../lehins.github.io/assets/iframes/2020-02-12-parallel-any-all/late-no-termination.html
-->

That's pretty good, if you ask me! I personally was really happy to see those
number. Sequential version of `any` was slightly faster then the defacto library in Haskell
for flat arrays. The amazing part of this benchmark is that even on such a cheap function as
`even` parallelization improved the runtime by almost two and half.


Keep in mind though: spawning off threads in Haskell is cheap, but not free, so there is
visible overhead when the size of the array drops by an order of a 1000. Traversing the
array sequentially gets so cheap that doing so in parallel inverts the picture:

```haskell
arrayLateShort :: Array P Ix1 Int
arrayLateShort =
  computeAs P $
  concat' 1 [A.enumFromStepN Seq 1 2 7000, A.singleton 2, A.enumFromStepN Seq 1 2 999]
```

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2020-02-12-parallel-any-all/late-short-no-termination.html" style="overflow: hidden; width: 100%; height: 112px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
stack bench --ba '--output late-short-no-termination.html --match pattern any/LateShort --template-short /home/lehins/github/haskell-benchmarks/template-short.tpl' && mv late-short-no-termination.html ../../lehins.github.io/assets/iframes/2020-02-12-parallel-any-all/late-short-no-termination.html
-->

This is totally expected though. There is always a bit of overhead for bookkeeping, which is
the cost we are all ready to pay if we want better performance where it actually matters.


## Short-circuiting

Thus far I was quite happy with performance. Let's talk about terrible wdge cases. I am
aware of a pitfall in `massiv`, such that if I relocate this element with value `2` to a
more unfortunate place, for exampletowards the beginning of an array, then it won't be that
fine and dandy.



<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2020-02-12-parallel-any-all/early-no-termination.html" style="overflow: hidden; width: 100%; height: 112px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
stack bench --ba '--output early-no-termination.html --match pattern any/Early --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv early-no-termination.html ../../lehins.github.io/assets/iframes/2020-02-12-parallel-any-all/early-no-termination.html
-->


This is where short circuiting comes in. I simply never got around to implementing it. Not
for the sequential, nor the parallel implementation. A tangentially related fun fact: in the
world of cryptography, this is the exactly the behavior you want in order to prevent timing
attacks. Despite that, the target audience of `massiv` is not cryptographers, so that is a
wart I definitely want to get rid of.

## Fix the sequential "any"

This is the point where we need to dive into some code in order to understand how it works,
so it can be fixed.


`any` function can be implemented as left fold. Another pun :) In `massiv` I took an
approach of the same implementation for sequential and parallel folding, whith some trickery
that chooses which approach to take depending on the strategy supplied, for example `Seq` or
`Par`. This means for us that even the sequential version will be just as complex as the
parallel one, but considering it is the latter we want to ulimately fix, might as well just
get on with it. This was the most general parallelizable index aware left fold implementation
in `massiv` before this post:


```haskell
ifoldlIO ::
     (MonadUnliftIO m, Source r ix e)
  => (a -> ix -> e -> m a) -- ^ Index aware folding IO action
  -> a -- ^ Accumulator
  -> (b -> a -> m b) -- ^ Folding action that is applied to the results of a parallel fold
  -> b -- ^ Accumulator for chunks folding
  -> Array r ix e
  -> m b
ifoldlIO f !initAcc g !tAcc !arr = do
  let !sz = size arr
      !totalLength = totalElem sz
  results <-
    withScheduler (getComp arr) $ \scheduler ->
      splitLinearly (numWorkers scheduler) totalLength $ \chunkLength slackStart -> do
        loopM_ 0 (< slackStart) (+ chunkLength) $ \ !start ->
          scheduleWork scheduler $
          iterLinearM sz start (start + chunkLength) 1 (<) initAcc $ \ !i ix !acc ->
            f acc ix (unsafeLinearIndex arr i)
        when (slackStart < totalLength) $
          scheduleWork scheduler $
          iterLinearM sz slackStart totalLength 1 (<) initAcc $ \ !i ix !acc ->
            f acc ix (unsafeLinearIndex arr i)
  F.foldlM g tAcc results
```


Some disection would be appropriate at this point:

* `MonadUnliftIO` is a really cool and correct way of dealing with exceptions. For the
  purposes of this post it is not relevant so just think of `m` in there as regular `IO`.

* `Source r ix e` type class gives us a mapping from index `ix` to value `e` in the array
  `Array r ix e`, which in the above function is done with `unsafeLinearIndex`. It is just
  like regular indexing in `C`: if you get it wrong you'll get a segfault, but it is fast,
  since there are no bounds checking.
* Unlike regular `foldl :: (b -> a -> b) -> b -> t a -> b`, this on is not only monadic, but
  also has two folding functions and two accumulators, one of which is index aware:

  * `(a -> ix -> e -> m a)` - first folding function will be used 




<!-- PARALLEL: -->
<!-- , and since its result is monoidal, we  -->


Prime function for benchmarking:

https://byorgey.wordpress.com/2020/02/07/competitive-programming-in-haskell-primes-and-factoring/


