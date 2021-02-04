---
layout: page
title: New random interface
categories: [blog, dev]
tags: [haskell, prng]
---

# New random interface

This post is about the new cool features in `random-1.2.0`. Most notably performance
improvements and the new monadic stateful interface.

## History

* At some point in the 90s `Random` module came into being and was included with ghc for
  many years until about a decade ago. Latest known versions that were wired in
  with GHC:
  * `random-1.1` briefly in `ghc-8.4.2` and `ghc-8.2.2`
  * before that `<= random-1.0.0.3` in `ghc-7.0.4` and older.

* February 1999 - Original version made it into [Haskell 98
  report](https://www.haskell.org/onlinereport/random.html) and is included in
  [`haskell98`](https://hackage.haskell.org/package/haskell98) package.

* July 2005 - first ticket on ghc tracker
  [#427](https://gitlab.haskell.org/ghc/ghc/-/issues/427) about `Random.StdGen` being
  slow.

* November of 2007 -
  [`random-1.0.0.0`](https://hackage.haskell.org/package/random-1.0.0.0) is available on
  Hackage

* December 2019 - I wrote a blog post about how terrible `random`'s performance is:
  [Random benchmarks](https://alexey.kuleshevi.ch/blog/2019/12/21/random-benchmarks/). I
  also suggested at the end of that post:

  > If it were up to me, here are the improvements I would make:
  >
  >  * Switch random package to depend on splitmix, instead of the other way round. Get
  >    rid of the internal StdGen implementation in favor of SMGen (i.e. type StdGen =
  >    SMGen).
  >
  >  * Improve the implementation of each individual instance of Random
  >
  >  * Make it possible for implementers of custom generators to override default
  >    implementations of individual primitive types (i.e. customizable random function).
  >
  >  * Add an interface to random package for stateful generators, which could be used by
  >    mwc-random and others alike.

* February 2020 - Dominic Steinitz confirmed my benchmark results from the blogpost and
  ["called out my
  bluff"](https://github.com/lehins/haskell-benchmarks/issues/3#issuecomment-584280581)
  about improvements I'd make if "it were up to me". To be more precise he asked me if I'd be
  willing to collaborate on getting those issues with `random` resolved.

* February 2020 - June 2020: Dominic Steinitz, Leonhard Markert and myself worked really
  hard on implementing proposed solutions as well as solving any other important issues we
  could find.

* 23rd of June of 2020 - all the hard work paid off and culminated in
  [random-1.2.0](https://hackage.haskell.org/package/random-1.2.0) being released.

## Original interface

It is important for the rest of the post to understand the original interface, so here is
a short recap.

A type class for Pseudo Random Number Generator (PRNG) implementers:

```haskell
class RandomGen g where
  genRange :: g -> (Int, Int)
  next     :: g -> (Int, g)
  split    :: g -> (g, g)
```

In other words if you have an algorithm for a pure and possibly splittable PRNG you'd be
able to create an instance for this class. There are a handful of packages available that
provide data types with an instance for this class, but majority of them aren't really
maintained. The important part is that `random` historically came with the default
implementation `StdGen` that has an instance for that `RandomGen` class.

A type class for values that can be generated using any pure PRNG that provides `RandomGen`
instance:

```haskell
class Random a where
  randomR :: RandomGen g => (a, a) -> g -> (a, g)
  random  :: RandomGen g => g -> (a, g)

  randomRs :: RandomGen g => (a, a) -> g -> [a]
  randoms  :: RandomGen g => g -> [a]

  randomRIO :: (a, a) -> IO a
  randomIO  :: IO a
```

All basic types like `Bool`, `Char`, `Int`, etc. have an instance for this class.

Functions for creating and initializing `StdGen` also came bundled in the package:

* Creating `StdGen` from some seed: `mkStdGen :: Int -> StdGen`
* Initializing `StdGen` from system entropy, eg. current time: `getStdGen :: IO StdGen`

### Examples

Here is an example of how we'd able to use this interface to generate a random phone number:

```haskell
import System.Random
import Text.Printf

newtype AreaCode = AreaCode { unAreaCode :: Int }
  deriving (Eq, Show, Num)

-- | The North American Numbering Plan (NANP) phone
data Phone = Phone { phoneAreaCode :: AreaCode
                   , phoneLocalNumber :: Int
                   }

instance Show Phone where
  show Phone {phoneAreaCode, phoneLocalNumber} =
    printf "+1-%03d-%03d-%04d" (unAreaCode phoneAreaCode) phoneSuffix phonePostfix
    where
      (phoneSuffix, phonePostfix) = phoneLocalNumber `quotRem` 10000

randomPhone :: RandomGen g => [AreaCode] -> g -> (Phone, g)
randomPhone areaCodes g =
  let (i, g') = randomR (0, length areaCodes - 1) g
      (phoneLocalNumber, g'') = randomR (0, 9999999) g'
   in (Phone {phoneAreaCode = areaCodes !! i, phoneLocalNumber}, g'')
```

In order to create a random phone number we need to generate two random values:

* one is a random index for selecting an area code from supplied list
* another one is for the local part of the phone number

Further we can use it in ghci like so:

```haskell
λ> gen <- getStdGen
λ> let (tollFreePhone, gen') = randomPhone [800,833,844,855,866,877,888] gen
λ> tollFreePhone
+1-843-193-5130
λ> let (newMexicoPhone, _gen'') = randomPhone [505,575] gen'
λ> newMexicoPhone
+1-505-263-9928
```

We'll revisit this example with the newly added interface later in this post.

## Problems

If there were no problems this post would have never been written, so let's talk about
issues we've set out to tackle. I will be intentionally brief in the next two sections,
because they are quite involved and deserve their own posts. Instead I will focus mostly on
issues with the interface.

### StdGen

* Bad quality of randomness

* Generated only ~31bits of data at a time (`[1, 2147483562]` range)

* Terrible performance characteristics

* Splitting produced sequences that weren't independent

__*Solution:*__ switched to [`splitmix`](https://hackage.haskell.org/package/splitmix)
package for `StdGen` implementation.

In my previous blogpost about `random` I've already identified the fastest PRNG
implementation available in Haskell, which was `splitmix`. So, at that point all we had to
do was to verify its quality and that is exactly what we did. Leonhard even wrote a
[blogpost about it](https://www.tweag.io/blog/2020-06-29-prng-test/).

Implementing a whole new PRNG from scratch would have added a tremendous amount of work for
us, so big thanks to the author of `splitmix`, Oleg Grenrus, for providing a perfect
solution to this problem and working with us on inverting its dependecy on `random`.

Reasons for switching to splitmix:

* Fastest PRNG implementation in Haskell

* Very good quality of randomness (passes most tests, eg. Dieharder)

* Generates 64bits of random data in one iteration

* Splitting produces independent sequences

### Performance

* Previous implementation of `StdGen` was slow.

* Generation of **all** types went through `Integer`.

* `genRange` was a historical mistake that was needed for some PRNGs that produced values
  in unusual ranges

__*Solutions:*__

* Switched `StdGen` to `splitmix`, as it was already mentioned earlier.

* Used [bitmask with rejection technique](https://www.pcg-random.org/posts/bounded-rands.html)
  for generating values in custom ranges.

* Major redesign of `RandomGen` class:

  * Deprecated `genRange` and `next` in favor of `genWord64` and/or `genWord32`

  * Made generation of other bit widths customizable: `genWord[8|16|32|64]` and
    `genWord[32|64]R`.

  * Allowed customization of array generation with `genShortByteString`. More on this later.

New definition of `RandomGen` class (default implementations are omitted):

```haskell
class RandomGen g where
  {-# MINIMAL split,(genWord32|genWord64|(next,genRange)) #-}
  genWord8 :: g -> (Word8, g)
  genWord16 :: g -> (Word16, g)
  genWord32 :: g -> (Word32, g)
  genWord64 :: g -> (Word64, g)
  genWord32R :: Word32 -> g -> (Word32, g)
  genWord64R :: Word64 -> g -> (Word64, g)
  genShortByteString :: Int -> g -> (ShortByteString, g)
  split :: g -> (g, g)

  -- Deprecated:
  genRange :: g -> (Int, Int)
  next :: g -> (Int, g)
```

None of these functions need to be used directly and only affect PRNG implementers. Thus
regular users don't really need to worry about these redesigns and deprecations.

### Random class

When we talk about PRNGs we immediately assume that random values are generated in uniform
distribution, unless otherwise specified. However `Random` class is poorly suited for
practical purposes for such distribution.

Here are some examples showing the reasons why:

* `Integer` has infinitely many values, so only `randomR` makes sense because it is
impossible to generate an infinitely large value. In order to complete the instance
declaration of the class `random` function has to generate values in some arbitrary range
and 64bit range was selected purely for performance reasons.

* `Double`/`Float` try to represent real values, which have an infinite range, thus is a
problem for `random` function. To make matters worse, floating point values are limited in
the number of real values that they can represent (approx range is `-5.0e-324` to
`5.0e-324` for 64bits) and these representable values are not equidistant from each
other. In other words, really large values have very little precision and precise numbers
have to be small. All these facts combined with special values `+/-Infinity`, `NaN` and
`-0` makes uniform distribution impossible when we don't know the range. For this reason
`random` function generates values in a `[0, 1]` region.

  There are more caveates in floating point number generation, all of which are described in
package documentation. Some of them have not been completely solved yet, so there is still
more work to be done in this area.

* Custom types, eg. `Uuid` - it doesn't make sense to generate a uuid in a range, so
  `randomR` has to throw an error.

__*Solution:*__

Split this class into two concepts: `Uniform` and `UniformRange`.

```haskell
class Uniform a where
  uniformM :: StatefulGen g m => g -> m a

class UniformRange a where
  uniformRM :: StatefulGen g m => (a, a) -> g -> m a
```

Taking this approach makes it possible for `Integer`, `Float` and `Double` to get
instances for `UniformRange` class, but not for `Uniform`.

Here is an example from the other end of the spectrum, where `UniformRange` doesn't make
sense, while `Uniform` can get a very natural and succinct implementation:

```haskell
data RGB a = RGB { red   :: a
                 , green :: a
                 , blue  :: a
                 } deriving (Eq, Show)

instance Uniform a => Uniform (RGB a) where
  uniformM g = RGB <$> uniformM g <*> uniformM g <*> uniformM g
```

There were suggestions of deprecating `Random` class because these two classes together
are more correct, however in my opinion this would be a tremendous mistake at this point
because there is too much code out there in a wild that depends on that class. Instead we
agreed that it will be used for generating values in a "Standard" distribution, which can
be type specific and doesn't have any theoretical backing. Despite that it sounds
funky, similar approaches are taken in other programming languages.


You might notice that `Uniform` and `UniformRange` classes look quite a bit different from
`Random`, but in fact they are a lot more general. Not only they work in `IO` monad, thus
making `randomIO` and `randomRIO` redundant, but they also allow these two functions to be
concocted out of monadic versions for free:

```haskell
uniform :: (RandomGen g, Uniform a) => g -> (a, g)
uniformR :: (RandomGen g, UniformRange a) => (a, a) -> g -> (a, g)
```

And this leads us to the next section which describes another new class `StatefulGen` that
makes this and much more possible.

### Stateful monadic generators

One of the other major goals that we set for ourselves was to provide a unified interface
for mutable generators as well as the pure ones. As it turns out it was not an easy task
because mutable generators come in different flavors:

* State transformer monads - when pure generator is the actual state in `StateT`
* Mutable variables - when pure generator is stored in `STRef` or an `IORef`
* Generators that depend on a large mutable state that has to be stored in some data
  structure such as a mutable vector. Here are packages that provide such generators in
  Haskell: [`mwc-random`](http://hackage.haskell.org/package/mwc-random),
  [`pcg-random`](http://hackage.haskell.org/package/pcg-random),
  [`sfmt`](https://hackage.haskell.org/package/sfmt) and
  [`mersenne-random`](https://hackage.haskell.org/package/mersenne-random)

Let's dive deeper into each one of these mutable generator types and see when they are
useful and how they tie in with the new interface.

#### State transformer monads

Passing generator around is not only inconvenient, but it also does not compose very
well. The natural solution to this problem is to use either `StateT` monad from
[`transformers`](https://hackage.haskell.org/package/transformers) or `RandT` monad from
[`MonadRandom`](https://hackage.haskell.org/package/MonadRandom) package.

Let's go back to our phone example and first rewrite it with the help of pure `uniformR`:

```haskell
uniformPhone :: RandomGen g => [AreaCode] -> g -> (Phone, g)
uniformPhone areaCodes g =
  let (i, g') = uniformR (0, length areaCodes - 1) g
      (phoneLocalNumber, g'') = uniformR (0, 9999999) g'
   in (Phone {phoneAreaCode = areaCodes !! i, phoneLocalNumber}, g'')
```

And now compare it to a very similar version that uses `StateT` monad:

```haskell
import Control.Monad.State

uniformPhoneState :: RandomGen g => [AreaCode] -> g -> (Phone, g)
uniformPhoneState areaCodes =
  runState $ do
    i <- state $ uniformR (0, length areaCodes - 1)
    phoneLocalNumber <- state $ uniformR (0, 9999999)
    pure Phone {phoneAreaCode = areaCodes !! i, phoneLocalNumber}
```

They look quite similar, but in the latter approach we don't need to do all this
error-prone passing of the pure generator around. Not only are they similar, but their
performance characteristics are exactly the same and this approach has been available to us
since the 90s (usage of `randomR` vs `uniformR` is not really relevant here).

It's a perfect place to take a look at how these two approaches compare to the new
interface we've recently added:

```haskell
uniformPhoneM :: StatefulGen g m => [AreaCode] -> g -> m Phone
uniformPhoneM areaCodes gen = do
  i <- uniformRM (0, length areaCodes - 1) gen
  phoneLocalNumber <- uniformRM (0, 9999999) gen
  pure Phone {phoneAreaCode = areaCodes !! i, phoneLocalNumber}
```

Without knowing what `StatefulGen` is all about it isn't quite clear how exactly they
compare, but it is obvious that they look very similar. While avoiding going too deep into
the details at this point I can show you how to recover the original function:

```haskell
uniformPhoneStateGen :: RandomGen g => [AreaCode] -> g -> (Phone, g)
uniformPhoneStateGen areaCodes g = runStateGen g (uniformPhoneM areaCodes)
```

Note that the type signature, actual functionality and performance are still exactly the
same. Here is a sample run:

```haskell
λ> gen <- getStdGen
λ> uniformPhone [505, 575] gen
(+1-575-193-5130,StdGen {unStdGen = SMGen 7548616687617844002 9974470818915064659})
λ> uniformPhoneState [505, 575] gen
(+1-575-193-5130,StdGen {unStdGen = SMGen 7548616687617844002 9974470818915064659})
λ> uniformPhoneStateGen [505, 575] gen
(+1-575-193-5130,StdGen {unStdGen = SMGen 7548616687617844002 9974470818915064659})
```

#### Mutable variables in `IO` and `ST`

The next logical question that comes to mind is: Why do we need this new approach if it
works, feels and looks almost the same as the other two that we had for almost two
decades? My claim is that this new approach works not only with pure `RandomGen`
generators, but also with the true mutable ones as well!

This means that the function `uniformPhoneM` can now be used in `ST` and `IO` monads with
generators that rely on actual mutable variables and data structures. Here is an example
that demonstrates `STGenM` usage that is backed by an `STRef`:

```haskell
import Control.Monad.ST
import Data.STRef

uniformPhoneST :: RandomGen g => STRef s [AreaCode] -> STGenM g s -> ST s Phone
uniformPhoneST areaCodesRef stGen = do
  areaCodes <- readSTRef areaCodesRef
  uniformPhoneM areaCodes stGen

uniformPhone' :: RandomGen g => [AreaCode] -> g -> (Phone, g)
uniformPhone' areaCodes g = runSTGen g $ \stGen -> do
  areaCodesRef <- newSTRef areaCodes
  uniformPhoneST areaCodesRef stGen
```

Above example isn't terribly convincing or even useful, since it is slower than the ones
before it and can easily be imitated by `StateT g (ST s)` monad. However this is just a
tip of an iceberg and there other scenarios where `StateT` would be impossible. For
instance, if we add concurrency into the mix. Let's produce up to 5 random phone numbers
concurrently each in a separate thread:

```haskell
import Control.Concurrent.Async (replicateConcurrently)

uniformPhones :: RandomGen g => [AreaCode] -> g -> IO ([Phone], g)
uniformPhones areaCodes g = do
  (phones, AtomicGen g') <-
    withMutableGen (AtomicGen g) $ \atomicGen -> do
      n <- uniformRM (1, 5) atomicGen
      replicateConcurrently n (uniformPhoneM areaCodes atomicGen)
  pure (phones, g')
```

In this example we still use a pure random number generator, but now it is wrapped into an
`IORef` and is named `AtomicGenM`.

```haskell
λ> print . fst =<< uniformPhones [505, 575] (mkStdGen 2021)
[+1-575-093-5381,+1-505-941-5864,+1-575-441-6931,+1-575-790-1299,+1-505-803-8627]
```

It is not the most efficient approach, because it is better to split a generator into one
per thread, but when there is not much contention for a single generator, then it is a
perfectly viable solution. The important takeaway, however, is that we were able to use
the exact same `uniformPhoneM` function here as well.

#### StatefulGen and FrozenGen explained

`StatefulGen` looks like a monadic version of `RandomGen` class, but without any splitting.


```haskell
class Monad m => StatefulGen g m where
  {-# MINIMAL (uniformWord32|uniformWord64) #-}
  uniformWord32R :: Word32 -> g -> m Word32
  uniformWord64R :: Word64 -> g -> m Word64
  uniformWord8 :: g -> m Word8
  uniformWord16 :: g -> m Word16
  uniformWord32 :: g -> m Word32
  uniformWord64 :: g -> m Word64
  uniformShortByteString :: Int -> g -> m ShortByteString
```

There are four instances provided for this class out of the box:

```haskell
(RandomGen g, MonadState g m) => StatefulGen (StateGenM g) m
RandomGen g                   => StatefulGen (STGenM g s) (ST s)
(RandomGen g, MonadIO m)      => StatefulGen (IOGenM g) m
(RandomGen g, MonadIO m)      => StatefulGen (AtomicGenM g) m
```

Three of which we already saw in action and `IOGenM` is just like `AtomicGenM`, except it
is faster, but not thread-safe. It is also exactly like `STGenM`, but in `IO` instead of
`ST`. If we dig deep into implementation of the last three mutable generators, we'll see
that they are all based on `MutVar#`, so it only makes sense that they are very closely
related.

The first one, however, works in a totally different category of mutation, namely pure state
passing. The peculiar part about `StateGenM` type is that it is simply a proxy that
carries around the type of the pure generator being used, while the actual generator value
is passed around by some monad with `MonadState` instance. This also means that it will
compose well with other transformer monads with `mtl` library. Here is how we can use it
together with `MonadReader`, for instance:


```haskell
import Control.Monad.Reader

uniformPhoneReaderStateM :: (MonadReader [AreaCode] m, StatefulGen g m) => g -> m Phone
uniformPhoneReaderStateM gen = do
  areaCodes <- ask
  uniformPhoneM areaCodes gen

uniformPhoneMTL :: RandomGen g => [AreaCode] -> g -> (Phone, g)
uniformPhoneMTL = runState . runReaderT (uniformPhoneReaderStateM StateGenM)
```

One way or another all of these guys are technically mutable, which means that they also
have a frozen immutable counterpart:

* `data StateGenM g     =           StategGenM` has `newtype StateGen g  = StateGen g`
* `newtype STGenM g s   =   STGenM (STRef s g)` has `newtype STGen g     = STGen g`
* `newtype IOGenM g     =     IOGenM (IORef g)` has `newtype IOGen g     = IOGen g`
* `newtype AtomicGenM g = AtomicGenM (IORef g)` has `newtype AtomicGen g = IOGen g`

This is where `FrozenGen` comes in:

```haskell
class StatefulGen (MutableGen f m) m => FrozenGen f m where
  type MutableGen f m = (g :: Type) | g -> f
  freezeGen :: MutableGen f m -> m f
  thawGen :: f -> m (MutableGen f m)
```

It is the class that relates the immutable type with the mutable one and allows us to go
between the two. Besides drawing a clear line between mutable and immutable state of a
PRNG, it also lets us use this concept abstractly without knowing the type of generator
ahead of time. A good real life like example that comes to mind is when we expect an
immutable version being stored in a file for a later reuse. Let's pull in `serialise`
package to demonstrate such use case:

```haskell
import Codec.Serialise (Serialise(..), readFileDeserialise, writeFileSerialise)

withStoredGen :: (Serialise f, FrozenGen f IO) => FilePath -> (MutableGen f IO -> IO a) -> IO a
withStoredGen filepath action = do
  frozenGen <- readFileDeserialise filepath
  (result, frozenGen') <- withMutableGen frozenGen action
  writeFileSerialise filepath frozenGen'
  pure result
```

Note that we have not specified the actual generator yet, but we already implemented a
function that can restore it from file, use it to generate some random data and then store
it back to file. Immutability here is the key, because mutable types don't get many
classes designed for them, which is also the case with `Serialise` and all other
serialization libraries.

#### Generators with large mutable state

As mentioned earlier one of the motivations behind this new design of `StatefulGen` and
`FrozenGen` was making it possible to use generators that rely on a large mutable state,
such as one provided by `mwc-random` package. I would like to thank maintainer of that
package Aleksey Khudyakov for a lot of constructive criticism and inspiration for some of
the ideas during design of this interface, as well as his other invaluable contributions
during this endeavour.

Starting with `mwc-random-0.15.0.1` it is possible to use all of this functionality
described above, since it provides an instance for its `Gen` type:

```haskell
instance (s ~ PrimState m, PrimMonad m) => StatefulGen (Gen s) m where
```

and another one for its `Seed`:

```haskell
instance PrimMonad m => FrozenGen Seed m where
  type MutableGen Seed m = Gen (PrimState m)
```

Let's see how those instances fall into our previous examples. First, we'll need to add
`Serialise` instance for `Seed`, since it is not available out of the box, but it is quite
easy to define:

```haskell
import qualified Data.Vector as VU
import System.Random.MWC (Gen, Seed, fromSeed, toSeed)

instance Serialise Seed where
  encode = encode . fromSeed
  decode = (toSeed :: VU.Vector Word32 -> Seed) <$> decode
```

With all of the above in our scope we'll generate an initial `Seed` and write it to file
for later reuse by `withStoredGen` and `uniformPhoneM` functions:

```haskell
λ> import System.Random.MWC (createSystemSeed)
λ> seed <- createSystemSeed
λ> writeFileSerialise "example-seed.bin" seed
λ> :set -XTypeApplications
λ> withStoredGen @Seed "example-seed.bin" (uniformPhoneM [505,575])
+1-505-589-8192
```

### Random binary data

A naive approach to generate random binary data, a.k.a. `ByteString`, is to generate a
list of `Word8`s and pack it:

```haskell
randomByteStringNaive :: RandomGen g => Int -> g -> ByteString
randomByteStringNaive n = pack . take n . randoms
```

This approach is very inefficient for two reasons:

* In order to generate `Word8` we have to generate 64bits of random data, which means 56
  of perfectly good random bits are discarded.
* Intermediate list, if not fused will cause unnecessary allocations

__*Solution:*__ allocate a region of memory of a desired size and write 64bits at a time,
while making sure that machine's CPU endianness does not affect the outcome.

A proper efficient solution is not very straightforward to implement and an optimal solution
might differ between PRNGs, so we provided a default implementation and made it
customizable in `StatefulGen` and `RandomGen`. Some examples on how to use
`uniformByteStringM` with different generators:

* with `System.Random.MWC.Gen`:
```haskell
λ> seed <- createSystemSeed
λ> mwcGen <- thawGen seed
λ> B.unpack <$> uniformByteStringM 15 mwcGen
[128,26,32,201,239,231,140,112,96,208,48,17,24,135,98]
```

* with `System.Random.StdGen`:
```haskell
λ> B.unpack $ runStateGen_ (mkStdGen 2021) (uniformByteStringM 15)
[78,232,117,189,13,237,63,84,228,82,19,36,191,5,128]
```

### A few words on `RandomGenM`

Until now I've talked about how we can use this new stateful interface in different
scenarios, including one where we use it to define pure functions that work on `RandomGen`
(i.e. `uniform` and `uniformR`). All is great, except when working on it I thought that it
would be sad if we couldn't take advantage of some other historic features such as
splitting of PRNGs or using `Random` class with our new monadic stateful interface. So we
came up with `RandomGenM`:

```haskell
class (RandomGen r, StatefulGen g m) => RandomGenM g r m | g -> r where
  applyRandomGenM :: (r -> (a, r)) -> g -> m a
```

which works with all stateful generators that are backed by a pure PRNG. This
allowed us to define:

* `randomM :: (RandomGenM g r m, Random a) => g -> m a`
* `randomRM :: (RandomGenM g r m, Random a) => (a, a) -> g -> m a`

and most likely in the future will serve as a replacement interface for the single global
generator `randomIO` and `randomRIO`


## Stats

After 5 months of work:
* [266 commits](https://github.com/idontgetoutmuch/random/compare/v1.1...v1.2-proposal)
* 150 total pull requests with a [100 of them merged](https://github.com/idontgetoutmuch/random/pulls?q=is%3Apr+is%3Amerged).
* Closed 6 existing issues and partially addressed at least 3 more.
* Exploration in API design yielded a discovery of 1 bug in GHC:
  [#18021](https://gitlab.haskell.org/ghc/ghc/-/issues/18021)

## Summary of achievements

* The quality of generated random values got much better
* Astonishing performance improvement. It only took 15 years since it was first reported
  being slow.
* Interface has been expanded dramatically
* Amount of documentation was increased quite a bit
* Modern test and benchmark suites have been added
* Very little breakage, majority of the functionality was kept backwards compatible.

## Future plans

* Generic deriving for `Uniform` and `UniformRange` classes is almost done, some of it has
  already been released
* Further improvements to floating point number generation
* `Uniform` and `UniformRange` instances for more types from base.
* Possibly another class for generating complex data structures, eg. lists, arrays, trees, etc.
* Type safety for distinguishing splittable vs non-splittable generators
* Mutable generator that works in `STM`.

## Special thanks to

* Dominic Steinitz [@idontgetoutmuch](https://github.com/idontgetoutmuch)
* Leonhard Markert [@curiousleo](https://github.com/curiousleo)
* Aleksey Khudyakov [@Shimuuar](https://github.com/Shimuuar)
* Andrew Lelechenko [@bodigrim](https://github.com/bodigrim)
* Daniel Cartwright [@chessai](https://github.com/chessai)
* Oleg Grenrus [@phadej](https://github.com/phadej)
* Richard Eisenberg [@goldfirere](https://github.com/goldfirere)
* Mathieu Boespflug [@mboes](https://github.com/mboes)
