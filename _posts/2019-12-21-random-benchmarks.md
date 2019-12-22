---
layout: page
title: Random Benchmarks
categories: [blog, dev]
tags: [haskell, rng, benchmarks]
---

# Random libraries and benchmarks

Benchmarks of random libraries, or how fast can we generate random numbers in Haskell. In
this post I will share with you my investigation into performance of libraries that can be
used for random number generation. Besides performance, I will provide a brief introduction
on how to use those libraries and what are their conceptual differences.

## Categories of randomness

When people talk about random values they immediately think of two possible categories:

1. Cryptographically secure pseudo random number generators, which are provided by such
   libraries as [`cryptonite`](https://www.stackage.org/package/cryptonite){:target="_blank"} or
   [`crypto-api`](https://www.stackage.org/package/crypto-api){:target="_blank"} and are used for things
   like encryption. __These are NOT the kind of libraries I will be talking about in this
   post__. They do deserve their own separate article and maybe some other Haskeller with
   an interest in that domain can whip one up.

2. Random number generators (RNGs) that are used for modeling, simulations, generative
   testing, gaming, blinking lights, what have you. __Such generators are exactly the kind
   that interest us today__.

The latter category employs various algorithms to achieve different degree of
randomness. Some produce better values than the others, which might make them differ in
unpredictability, longer or shorter periods, or even suitability for certain data types but
not the others. They might pass or fail [diehard
tests](https://en.wikipedia.org/wiki/Diehard_tests){:target="_blank"} and have all sorts of other properties
that such algorithms are expected to possess. You can study each individual one at your
leisure, this is not gonna be the point of this post. It is important, though, that you
do understand the quality of the algorithm being used before jumping into conclusions
about the performance of its implementation. Just to give an example a
[Xorshift](https://en.wikipedia.org/wiki/Xorshift){:target="_blank"} algorithm is known to be fast, but
unreliable, when compared to others.

One other crucial point I would like to make is that I will omit any subjective
remarks about the quality of the code in all libraries discussed as well as the correctness of
implementation of claimed algorithms. In other words, even if a package is a total phony,
all I care about today is performance, so you be the judge of the rest.

## Plethora of libraries

There are quite a few non-cryptographic random generators available on Hackage and
Stackage, which implement different algorithms with similar APIs. All of them can be
placed into four categories:

### Pure RNGs

This is the kind of generator that produces a new random value together with another
instance of the same generator, which can further be used to generate another value
(possibly of different type) and another generator... and so on and so forth, until you get
your initial generator and an extremely long cycle starts repeating itself.

Great thing about such generators is that not only they can also be used in any pure
functions, but they are also 100% reproducible, granted that the initial generator was
created from the same seed. This great property comes from the fact that they rely on a
small immutable state that can simply be copied with slight changes, thus making them pure
and "stateless". Of course you can thread a generator in a `StateT` like monad or store it
in `IORef`, but that is not a requirement.

The most popular package that is equipped with such functionality in Haskell is
[`random`](https://hackage.haskell.org/package/random){:target="_blank"}. I think it is still popular only
because it was the first of its kind in Haskell and is the defacto library when people are
learning about pure random number generation. Keep reading further though to realize, if
you haven't yet, that there are much better packages available. Nevertheless let's look at
an example:

```haskell
λ> import System.Random
λ> g0 <- newStdGen
λ> let (x, g1) = next g0
λ> let (y, _) = next g1
λ> print y
608110287
λ> print x
1021702771
λ> print $ fst $ next g0
1021702771
```

Above example depicts a critical property of pure generators: applying `next` to `g0` will
always produce the same value.

### Splittable pure RNGs

Just as before, we can generate values in a pure setting, but now we can also split a
generator into two new distinct ones. This is a great property to have in a pure RNG,
because it allows us to produce deterministic sequences of random data in a concurrent or
even distributed environment. Let's see a splittable generator in action:

```haskell
λ> import System.Random
λ> import System.Random.SplitMix (mkSMGen)
λ> let g0 = mkSMGen 123456789
λ> let (g01, g02) = split g0
λ> :i randomR
class Random a where
  randomR :: RandomGen g => (a, a) -> g -> (a, g)
  ...
  	-- Defined in ‘System.Random’
λ> fst $ randomR (0, 1000) g01 :: Word
553
λ> fst $ randomR (0, 1000) g02 :: Word
410
λ> fst $ randomR (0, 1000) g01 :: Word
553
```

`split` function allows us to create two new distinct generators from `g0`, both of which
we could pass to two separate pure functions further down in the call tree. Moreover,
nothing prevents us from passing both of those `g01` and `g02` generators across the
wire to some other two machines. This would, as with any other RNG, let everyone know the
next random value each of the generators could produce. The power comes from the fact that
when paired with a pure functions, that can use those generators, all three of the
machines (including the sender) will be able to know deterministically the full
sequence of values each of them can produce. Try the above code on your computer and you will
get the same values.

We are going to exploit the splittable property of generators to produce deterministic
sequences of random numbers in parallel on multiple cores.

Please note that I used the same `randomR` and `split` functions on the `SMGen` generator
from `splitmix`. That's right, all of the pure generators in Haskell can use the same
interface from `random` package, more specifically `RandomGen` and `Random` classes. I
think that is another explanation for such popularity of that package.

Not all pure generators are splittable though, for example `mersenne-random-pure64`
package does not fall into that category, despite being pure:

```haskell
λ> import System.Random.Mersenne.Pure64
λ> import System.Random
λ> mtGen <- newPureMT
λ> fst $ randomR (0, 1000) mtGen :: Word
939
λ> split mtGen
*** Exception: System.Random.Mersenne.Pure: unable to split the mersenne twister
```

One might argue, that it is always possible to split a generator simply by creating a
random value and using it as a seed for the second generator. Such an approach might even
be OK if you don't care about how good your random numbers are. In fact, I've
personally abused a similar approach a few times during random property testing, but from
what I gather, unless the algorithm directly supports splitting, you will not get the same
quality of random numbers as it is actually promised by the algorithm.

### Stateful RNGs

Generators in the previous section had the luxury of being very small in size, despite being
mathematically complex. This fact allowed them to not rely on mutation to perform
efficiently, which in turn made them pure. RNGs in this section have to carry a large
mutable array in their state around, a portion of which is mutated each time a new random
value is generated. One such popular package is
[`mwc-random`](https://www.stackage.org/package/mwc-random){:target="_blank"}. Because there is mutation
happens behind the scenes, we have to live inside of
[`IO`](https://downloads.haskell.org/~ghc/8.8.1/docs/html/libraries/base-4.13.0.0/System-IO.html#t:IO){:target="_blank"},
[`ST`](https://downloads.haskell.org/~ghc/8.8.1/docs/html/libraries/base-4.13.0.0/Control-Monad-ST.html#t:ST){:target="_blank"}
or some transformer on top of one of those two that implements a
[`PrimMonad`](https://www.stackage.org/haddock/lts-14.17/primitive-0.6.4.0/Control-Monad-Primitive.html#t:PrimMonad){:target="_blank"}
instance. Here is how it works in ghci (which is inherently `IO`):


```haskell
λ> import System.Random.MWC
λ> gen <- createSystemRandom
λ> uniformR (0, 1000) gen :: IO Word
602
λ> uniformR (0, 1000) gen :: IO Word
963
```

As you can see, despite that we pass the same generator as an argument to `uniformR`, we
get different values back. This is because our generator is mutated behind the scenes in
the `IO` monad.

One cool thing is that regardless of the fact that out generator is inherently effectful, we
can still encapsulate it in a pure computation with `ST` monad. All we need to do is pass the
seed for a previously created generator and we will get a reproducible result. Here is an
example of a randomly generated list of values within a pure function that uses a safe
mutable interface underneath:

```haskell
import Control.Monad (replicateM)
import Control.Monad.ST (runST)
import System.Random.MWC (restore, uniform)

randomList :: Variate a => Seed -> Int -> [a]
randomList seed n =
  runST $ do
    gen <- restore seed
    replicateM n (uniform gen)
```


```haskell
λ> sysSeed <- save =<< createSystemRandom
λ> randomList sysSeed 4 :: [Float]
[0.5644637,0.9203384,0.30062655,8.7106675e-2]
λ> randomList sysSeed 5 :: [Float]
[0.5644637,0.9203384,0.30062655,8.7106675e-2,0.5830376]
```

Here we can observe that the same seed results in exactly the same values, as one would
expect from a pure computation. This approach does not come without a drawback: you don't
want to generate small amounts of data this way, because saving/restoring the seed is an
expensive operation, when compared to a single random number generation.

If you are running your computation in `IO`, you might be tempted to use the same
generator between multiple threads. Don't! There is no locking in place to prevent
corruption of the state. Not all is lost though. The only thing that is required for
generation of random values on multiple cores is the preparation of separate instances of
RNGs prior to parallelization. Even though they are stateful, they can safely function
independently, which is even better than locking, since it avoids contention for the same
resources. This will be important in our parallel benchmarks.

### Utilizers of RNGs

The last category is the most diverse one, it is all the libraries out there that use
uniform random number generators to produce some other random data:

* Wrappers (or built-in) that have functionality for producing a whole variety of
  distributions: [`random-fu`](https://hackage.haskell.org/package/random-fu){:target="_blank"},
  [`mwc-random`](https://hackage.haskell.org/package/mwc-random){:target="_blank"},
  [`statistics`](https://hackage.haskell.org/package/statistics){:target="_blank"}, etc.
* Wrappers around RNGs that provide different interface, for example monadic:
  [`MonadRandom`](https://hackage.haskell.org/package/MonadRandom){:target="_blank"},
  [`random-source`](https://www.stackage.org/package/random-source){:target="_blank"},
* Libraries that shuffle elements of data structures by using RNGs:
  [`random-shuffle`](https://www.stackage.org/package/random-shuffle){:target="_blank"}
* Libraries that do randomized property testing:
  [`QuickCheck`](https://hackage.haskell.org/package/QuickCheck){:target="_blank"},
  [`hendgehog`](https://hackage.haskell.org/package/hedgehog){:target="_blank"},
  [`genvalidity`](https://hackage.haskell.org/package/genvalidity){:target="_blank"}, etc.
* Libraries that generate realistic looking random data:
  [`fakedata`](https://hackage.haskell.org/package/fakedata){:target="_blank"},
  [`fake`](https://hackage.haskell.org/package/fake){:target="_blank"},
  [`lipsum-gen`](https://hackage.haskell.org/package/lipsum-gen){:target="_blank"}, etc.
* Whatever else you can think of that uses random numbers

Point is, once we have a uniform random number generator, we can simulate all other random
stuff. Maybe someday I'll get a chance to compare the performance of libraries that
produce numbers in other distributions, but not today.

## Benchmarks

Special note on system random generators, such as `getrandom()`, `/dev/random`,
`/dev/urandom`, `CryptoAPI` etc. Those RNGs are usable over FFI from Haskell, and if my
goal was to compare implementation across languages, they would serve nicely as a
baseline. That's not our purpose today, so we'll skip the system generators, especially
since the bindings I've tried were pretty slow.

### Motivation

Earlier this year I stumbled upon Ben Gamari's question on StackOverflow about trying to
generate an array with random numbers in parallel, which I was [really happy to
answer](https://stackoverflow.com/questions/11230895/parallel-mapm-on-repa-arrays/56449011#56449011){:target="_blank"},
since I had the exact solution for it in
[`massiv`](https://github.com/lehins/massiv){:target="_blank"}. Naturally, I wanted to benchmark my
solution. Once I wrote the benchmarks, my curiosity took me deeper into the realm of
randomness. So in a way, that question is the cause for this blog post. Thanks, Ben! ;)

### Method

In order to properly benchmark random number generators I could have went in a few
directions. One would be creating manual loops that generate a whole lot of random numbers
and discard them right away. Alternatively, I could have generated lists of random
numbers, like in one of the examples earlier. Instead, I went for unboxed arrays for a few
important reasons. First of all, I already had an efficient solution for generating arrays
using RNGs sequentially and in parallel with
[`randomArray`](https://www.stackage.org/haddock/lts-14.17/massiv-0.4.4.0/Data-Massiv-Array.html#v:randomArray){:target="_blank"}
and
[`randomArrayWS`](https://www.stackage.org/haddock/lts-14.17/massiv-0.4.4.0/Data-Massiv-Array.html#v:randomArrayWS){:target="_blank"}
functions using pure and stateful generators respectfully. Secondly, a flat unboxed array
is the most efficient way to store large number of values, be they random or not. Most
importantly though, as long as we are using the same approach for benchmarking all of the
libraries, we are still being fair, regardless that using some other data structure or
none at all could have produced slightly different results.

Both of these functions are pretty well documented, but to give you the gist of it: the
first one allows generating numbers purely, relying on splitability for parallelization,
while the second one runs in `IO` and pre-creates separate generators for each worker
thread in order to do proper parallelization.

### Libraries with pure RNGs

A list of libraries, which I was able to find that provide a pure interface for random
number generation, together with their versions that were used in the benchmarks
below. Libraries are sorted in the order of their appearance (i.e. initial version
uploaded on hackage).

|:-------:|:-----------------:|:----------------:|:--------------:|
| Package | First appeared on | Last released on | Latest release |
| [`random`](https://hackage.haskell.org/package/random){:target="_blank"} | 2007-11-03 | 2014-09-16 | `random == 1.1` |
| [`mersenne-random-pure64`](https://hackage.haskell.org/package/mersenne-random-pure64){:target="_blank"} | 2008-01-28 | 2016-08-29 | `mersenne-random-pure64 == 0.2.2.0` |
| [`xorshift`](https://hackage.haskell.org/package/xorshift){:target="_blank"} | 2011-01-05 | 2011-04-11 | `xorshift == 2` |
| [`AC-Random`](https://hackage.haskell.org/package/AC-Random){:target="_blank"} | 2011-08-25 | 2011-08-25 | `AC-Random == 0.1` |
| [`Random123`](https://hackage.haskell.org/package/Random123){:target="_blank"} | 2013-05-04 | 2015-03-20 | `Random123 == 0.2.0` |
| [`tf-random`](https://hackage.haskell.org/package/tf-random){:target="_blank"} | 2013-09-18 | 2014-04-09 | `tf-random == 0.5` |
| [`pcg-random`](https://hackage.haskell.org/package/pcg-random){:target="_blank"} | 2014-12-15 | 2019-05-18 | `pcg-random == 0.1.3.6` |
| [`Xorshift128Plus`](https://hackage.haskell.org/package/Xorshift128Plus){:target="_blank"} | 2015-04-14 | 2015-04-14 | `Xorshift128Plus == 0.1.0.1` |
| [`pcgen`](https://hackage.haskell.org/package/pcgen){:target="_blank"} | 2017-04-29 | 2017-07-05 | `pcgen == 2.0.1` |
| [`splitmix`](https://hackage.haskell.org/package/splitmix){:target="_blank"} | 2017-06-30 | 2019-07-30 | `splitmix == 0.0.3` |

Every single one of the above libraries provides an instance for
[`RandomGen`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#t:RandomGen){:target="_blank"}
class, so we are going to try and use it. `AC-Random` and `Xorshift128Plus` did not have such
an instance implemented, but they had the right helper functions to make it possible for
me to provide one.

We will start by disqualifying one package, namely `Random123`, and put it on a wall of
shame as being by far the slowest of them all (`random` package is only listed here as a
baseline):

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/disqualified.html" style="overflow: hidden; width: 100%; height: 100px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output disqualified.html --match pattern Disqualified --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv disqualified.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/disqualified.html
-->

Here is a benchmark for one of `RandomGen` functions `next :: g -> (Int, g)` which generates a random
`Int` and a new generator. Sequential and parallel generations can be distinguished by
different colors:

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/next.html" style="overflow: hidden; width: 100%; height: 532px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output next.html --match pattern Pure/next --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv next.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/next.html
-->

You might notice that `mersenne-random-pure64`, `xorshift`, `AC-Random` and
`Xorshift128Plus` do not have parallel benchmarks. That is because, as discussed earlier,
they do not provide any facilities for splitting the generator, which disqualifies them
from concurrent deterministic random number generators.

Also, note that `pcg-random` has two different pure implementations one regular and
another one experimental called "fast", both of which are written in pure Haskell. It also
has stateful implementation of regular and "fast", which we are gonna skip for now and
look at in the next section.

Now, I'd like to step back and try to understand what was exactly that we just
benchmarked. In particular, what was the possible value of `Int` we were getting for each
of the generators. To answer that question, we'll have to look at another `RandomGen`'s
function: `genRange :: g -> (Int, Int)`. I made a little helper function that goes through
all generators and prints the ranges for possible values those generators could produce:

```haskell
λ> printRanges
[                   1,           2147483562] - random:System.Random.StdGen
[-9223372036854775808,  9223372036854775807] - mersenne-random-pure64:System.Random.Mersenne.Pure64.MTGen
[         -2147483648,           2147483647] - xorshift:Random.Xorshift.Int32.Xorshift32
[-9223372036854775808,  9223372036854775807] - xorshift:Random.Xorshift.Int64.Xorshift64
[                   0,           4294967295] - AC-Random:Random.MWC.Primitive.Seed
[                   0,           2147483562] - tf-random:System.Random.TF.TFGen
[-9223372036854775808,  9223372036854775807] - pcg-random:System.Random.PCG.Pure.SetSeq
[-9223372036854775808,  9223372036854775807] - pcg-random:System.Random.PCG.Fast.Pure.FrozenGen
[-9223372036854775808,  9223372036854775807] - Xorshift128Plus:System.Random.Xorshift128Plus.Gen
[         -2147483648,           2147483647] - pcgen:Data.PCGen.PCGen
[-9223372036854775808,  9223372036854775807] - splitmix:Data.SplitMix32.SMGen
[-9223372036854775808,  9223372036854775807] - splitmix:Data.SplitMix.SMGen
```

When we look at the ranges, we might suspect that the earlier benchmark did not treat
these generators equally fair, and that suspicion would be totally justified. On the other
hand though, those ranges give us a better idea about the performance of a particular
generator. For example `AC-random` or `pcgen` can generate in one go a 32bit random
number, which means that in order to generate a uniformly distributed `Int64`, all we'd
have to do is call the generator twice. So, let's actually be fair and benchmark
generation of the full 64 bits.

Good thing for us is that `random` package is versatile enough and provides us with a
[`Random`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#t:Random){:target="_blank"}
type class, which is designed to use any of the above generators in order to produce a
uniformly distributed value in the full range of a type that has such instance
implemented. This means that all we need to do is call [`random :: RandomGen g => g -> (a,
g)`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#v:random){:target="_blank"}
function and we will get a uniformly distributed value in a predefined range, which for
bounded types like `Word64` it will be the full range.

The only change I made to the above benchmarks was I swapped `next :: RandomGen g => g ->
(Int, g)` function for a type restricted `random :: RandomGen g => g -> (Word64, g)`:

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/random64.html" style="overflow: hidden; width: 100%; height: 532px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output random64.html --match pattern Pure/random/Word64 --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv random64.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/random64.html
-->

__Wait! What?__ What happened there? All of a sudden every single one of our generators
started to perform x10 - x50 times slower. As I mentioned just a couple of paragraphs ago,
it would have been expected for some generators to get a factor of x2 slow down, since
they can't generate 64bits in one go. Therefore the results that we got can't possibly be
explained by this, so what is really causing such a huge regression? As it turns out,
[`randomR`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#v:randomR){:target="_blank"}
and consequently
[`random`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#v:random){:target="_blank"}
are oblivious to the concrete type that they are generating and all of the integral
numbers go through the arbitrary precision `Integer` in order to produce the value in a
desired range. Simply put, whatever number of bits you want to generate, be it 8 or 64 it
will be just as slow.

I personally wouldn't care much if it was some sporadically used function, but I can't
emphasize enough the impact of this. I see both of these functions `random` and `randomR`
used extensively in the Haskell community. I will even go on a limb and say that we all
use those two indirectly on a daily basis. Just to give you an idea, both `QuickCheck` and
`Hedgehog` uses those functions, which means everyone uses them, since we all writes
generative test, right? ;)

__The worst implication of the ridiculously slow implementation of `randomR` is, that
regardless of how fast your generator really is, you will always get terrible
performance!__ This fact is clearly depicted in the benchmarks above.

For the sake of argument, let's take a look at what performance of producing a `Word64`
can be for each of the generators, if we bypass the `Random` class. Some packages provide
built-in functions for generating `Word64`, which I happily used, but for the ones that do
not have such functionality out of the box, I've written the two functions below, choice
of which one depended on the generator's range:

```haskell
random32to64 :: RandomGen g => g -> (Word64, g)
random32to64 g0 = ((fromIntegral hi `unsafeShiftL` 32) .|. fromIntegral lo, g2)
  where
    (hi, g1) = next g0
    (lo, g2) = next g1

random64to64 :: RandomGen g => g -> (Word64, g)
random64to64 g0 = (fromIntegral i64, g1)
  where
    (i64, g1) = next g0
```

Both `tf-random` and `random` generate slightly less than 31 bits of entropy in one
iteration, instead of 32 or 64 as it is with others, so the simple approach from above
will not work, therefore I will just exclude `tf-random` and will keep `random`'s slow
implementation as the baseline for comparison:

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/custom.html" style="overflow: hidden; width: 100%; height: 476px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output custom.html --match pattern Pure/custom/Word64 --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv custom.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/custom.html
-->

It is clear now as night and day, that `random` function is the culprit and should be
either fixed or avoided when possible.

In order to get a better picture of the libraries that do perform well I need to get rid
of the slow ones, therefore I am now forced to drop `random`, `AC-Random`, `pcgen`,
`splitmix` (only the 32bit version). Note, that there is a disclaimer in the `splitmix`'s
haddock not to use the `SplitMix32`, which I included thus far for completeness only.


<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/fast.html" style="overflow: hidden; width: 100%; height: 280px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output fast.html --match pattern Pure/fast/Word64 --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv fast.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/fast.html
-->



From a quick look at all the benchmarks above, the clear winner is
[`splitmix`](https://www.stackage.org/package/splitmix){:target="_blank"}. Now let's move towards stateful
RNGs and see how they compare with pure generators.


### Libraries with stateful RNGs

Below is the list of packages that I was able to find, which provide RNGs that depend on a
mutable state. Same as before, packages are in the order of appearance and the versions
benchmarked are included as well:

|:-------:|:-----------------:|:----------------:|:--------------:|
| Package | First appeared on | Last released on | Latest release |
| [`mwc-random`](http://hackage.haskell.org/package/mwc-random){:target="_blank"} | 2009-12-26 | 2018-07-11 | `mws-random == 0.14.0.0` |
| [`sfmt`](https://hackage.haskell.org/package/sfmt){:target="_blank"} | 2014-12-09 | 2015-04-15 | `sfmt == 0.1.1` |
| [`pcg-random`](https://hackage.haskell.org/package/pcg-random){:target="_blank"} | 2014-12-15 | 2019-05-18 | `pcg-random == 0.1.3.6` |
| [`mersenne-random`](https://hackage.haskell.org/package/mersenne-random){:target="_blank"} | 2008-01-22 | 2011-06-18 | `mersenne-random == 1.0.0.1` |

In a very similar fashion as before I ran sequential and parallel generations of random
arrays, except now with stateful generators.

<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/io.html" style="overflow: hidden; width: 100%; height: 280px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output io.html --match pattern IO/Word64 --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv io.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/io.html
-->

The intriguing part of this chart for me was the fact that stateful implementations of
`pcg-random` and `mersenne-random` did not parallelize well at all. My gut feeling
suggests that it has something to do with `FFI` bindings and some peculiarities of
underlying `C` implementations. Although `sfmt` performed pretty well both sequentially
and in parallel, despite being pretty much all written in `C`.

## Finalists

In the end I would like to combine the finalists of this performance race from all of the
three categories: pure, pure splittable and stateful. Packages are now sorted by the
median of duration of their runtimes:


<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/finalists.html" style="overflow: hidden; width: 100%; height: 420px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output finalists.html --match pattern Finalists/Word64 --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv finalists.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/finalists.html
-->

An extra bonus benchmarks for the libraries that could efficiently generate `Double`.
I’ve tried
[`System.Random.random`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#v:random){:target="_blank"}
function first and as I suspected it performs just as awfully for `Double`s as for
integral types, therefore amongst pure generators I kept only the ones that provided
functions for `Double` out of the box.



<iframe id="discussion_iframe" frameborder="0" name="discussion" src="/assets/iframes/2019-12-21-random-benchmarks/finalists-double.html" style="overflow: hidden; width: 100%; height: 280px;" scrolling="auto">Here should be benchmarks</iframe>

<!--
$ stack bench --ba '--output finalists-double.html --match pattern Finalists/Double --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv finalists-double.html ../../lehins.github.io/assets/iframes/2019-12-21-random-benchmarks/finalists-double.html
-->


## Conclusion

I started with a goal of finding a goto library for myself to use, whenever I would need
random numbers. I certainly found what I was looking for and since all of us need a bit of
randomness once in a while, I thought I'd share these findings with all of other Haskellers
out there. Hope you find them useful.

In the process of researching and experimenting I made a few realizations for myself, thus
would like to share those with you as well, despite that they might be a bit subjective:

* [`splitmix`](https://hackage.haskell.org/package/splitmix){:target="_blank"} pleasantly surprised me with
  its performance. Great job, Oleg!  Parallelization of the algorithm isn't that great,
  only a factor of x2 on a quad core CPU, but I suspect it might be because it is getting
  bounded by the speed of my memory, rather than the algorithm or the implementation. This
  probably means that it is already pretty close to the optimal performance, although, as
  noted in the haddock, a possibility of future addition of SIMD to GHC (and `splitmix`)
  could give it a slight boost.

* [`sfmt`](https://hackage.haskell.org/package/sfmt){:target="_blank"} turned out to be a gem I never knew
  about. Underneath it relies on implementation in `C` and SIMD instructions, so it might
  not be extremely portable, but that doesn't mean it can't be useful for a whole variety
  of projects, especially since it turned out to be the fastest stateful generator.

* [`mwc-random`](https://hackage.haskell.org/package/mwc-random){:target="_blank"} has been around the block
  and is quite a popular library. It didn't turn out to be the fastest, but it still
  performs quite well and running in parallel improves the runtime significantly. Moreover,
  the interface is solid and provides ability to generate other distributions out of the
  box.

* [`random`](https://hackage.haskell.org/package/random){:target="_blank"} is the package I was quite
  disappointed in. I was more than sure that `StdGen` will have poor performance, but I
  did not expect
  [`Random`](https://hackage.haskell.org/package/random-1.1/docs/System-Random.html#t:Random){:target="_blank"}
  class instances to be so slow. I know that there [was some effort in
  improving](https://www.google-melange.com/archive/gsoc/2015/orgs/haskell/projects/nkartashov.html){:target="_blank"}
  it, but as it turns out [that
  effort](https://gist.github.com/nkartashov/e46fd146b1df2d79aaf3){:target="_blank"} was not only
  unfruitful, but it was also directed at the wrong problem. From the looks of it the true
  problem is not in the generator that is being used in the library, but in the way that all
  generators are being used. Of course, it doesn't mean the `StdGen` is not a problem, it's
  just that problem today has an easy fix.


If it were up to me, here are the improvements I would make:

* Switch `random` package to depend on `splitmix`, instead of the other way round. Get rid
  of the internal `StdGen` implementation in favor of `SMGen` (i.e. `type StdGen =
  SMGen`).
* Improve the implementation of each individual instance of `Random`
* Make it possible for implementers of custom generators to override default
  implementations of individual primitive types (i.e. customizable `random` function).
* Add an interface to `random` package for stateful generators, which could be used by
  `mwc-random` and others alike.

Thank you. If you'd like to inspect the code and convince yourself of legitimacy of the
discussed benchmarks by running them yourself or scrutinizing them some other way, they
can be found in my
[haskell-benchmarks](https://github.com/lehins/haskell-benchmarks/tree/master/random-benchmarks){:target="_blank"} repo.
