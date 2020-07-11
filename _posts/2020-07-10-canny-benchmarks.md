---
layout: page
title: Performance of Haskell Array libraries through Canny edge detection
categories: [blog, dev]
tags: [haskell, rng, benchmarks]
---

# Performance of Haskell Array libraries through Canny edge detection

## Intro

How many implementations of Canny edge detection do we have in Haskell? How fast are they
and which array libraries are responsible for their performance? These are the questions
I'll try to answer in this blog post.

## Preface

I am currently working on getting Haskell Image Proceesing library, aka
[HIP](https://github.com/lehins/hip), to be faster, better, friendlier and more correct
than it is today. The biggest change is the rewrite with
[`massiv`](https://github.com/lehins/massiv) library instead of current implementation that
uses [`repa`](https://github.com/haskell-repa/repa) and
[`vector`](https://github.com/haskell/vector). One of the algorithms I'd like to add in
the process of working on this transition is [Canny edge
detection](https://en.wikipedia.org/wiki/Canny_edge_detector). Which also seems like a
great algorithm to use for comparing performance of libraries that can do array
processing in Haskell, so that's what I did and that is exactly what this blog post is
about.


## Implementations

In pure Haskell I only found two implementations of Canny edge detection algorithm. If
there is one that I missed, please let me know. One was written by [Ben
Lippmeier](https://github.com/benl23x5) for the
[`repa`](https://hackage.haskell.org/package/repa) project and another one by [Raphael
Javaux](https://github.com/RaphaelJ) for his image processing library
[`friday`](https://hackage.haskell.org/package/friday). That would be sad if that was all
to it, but there are at least three other implementations that I know of, which all seem
to be modified versions of Ben's implementation that work for other libraries, including
mine. This is really great, because we have very similar looking implementations of the
same algorithm that use totally different libraries and produce nearly identical
results. This allows me to write reliable and objective benchmarks, because I personally
didn't take part in neither of those implementations, except the variant for `massiv`, of
course, since that is the library I authored.

I only care about the code that runs on CPU, therefore GPU is completely outside of the
scope of this post. Below is the list of implementations that I found:

* [`repa`](https://github.com/haskell-repa/repa/blob/0eb7373d61ca6f0bffeb10d605547d9e5048ab00/repa-examples/examples/Canny/src-repa/Main.hs) - many thanks to Ben Lippmeier for writing it.
* [`accelerate`](https://github.com/AccelerateHS/accelerate-examples/tree/a78f3c001e2375e0cb49d8a8a86d0af0b388b95d/examples/canny/src-acc) - thanks to Trevor L. McDonell
* [`yarr`](https://github.com/lehins/yarr/blob/4167330ec947c84c48ed5d105bc3aac83fa9db8c/tests/canny.hs) - thanks to Roman Leventov
* [`friday`](https://github.com/RaphaelJ/friday/blob/194d99a5f660cf1e0ea3fd39a066cf157c6efaf5/src/Vision/Detector/Edge.hs) - thanks to Raphael Javaux


There are two other Canny edge detection implementations in other languages that are
available in Haskell by the means of FFI:
[`opencv`](https://hackage.haskell.org/package/opencv-0.0.2.1/docs/OpenCV-ImgProc-FeatureDetection.html#v:canny)
and
[`arrayfire`](https://hackage.haskell.org/package/arrayfire-0.6.0.0/docs/ArrayFire-Image.html#v:canny). I'd
really like to have benchmark comparison for them as well, but for now my focus was on
pure Haskell.

## Array Libraries

Few personal notes on the libraries in question:

* [`massiv`](https://hackage.haskell.org/package/massiv) is the library that I wrote,
  which is a personal hobby that I picked up a few years ago and therefore it is always in
  active development. Naturally, I am very biased towards the library, so feel free to take
  that into consideration, but note that with respect to performance analysis I did my
  best to make the set up as fair as possible to all participants.
* [`accelerate`](https://hackage.haskell.org/package/accelerate) is also being active
  worked on. There is a whole research team from Ultrech University that is behind
  it. Constant improvements have a slightly negative affect on examples, which tend to lag
  behind a bit and prevented me from building all the necessary dependencies against
  master branch, but the newest version on hackage worked just fine. Important thing to note
  about this library is that its primary focus is to run Haskell code on GPUs, which is
  not something that I am utilizing here. The huge downside of this library in my opinion
  is that even if you only want your program to run on CPU you are still forced to use
  LLVM, which is not always trivial to setup. Another personal dislike I have is that it
  requires you to use a special DSL for the code you write, but it is a small price to pay
  for ability to run Haskell on a GPU.
* [`repa`](https://hackage.haskell.org/package/repa) seems to be in a very inactive mode
  with a few maintainers occasionally accepting patches and updating bounds of
  dependencies, but no new features are being developed. The initiative on the rewrite of
  next version of repa-4 a few years ago has stalled and is now bitrotting in the
  graveyard of unfinished projects.
* [`yarr`](https://hackage.haskell.org/package/yarr) has been abandoned for a few years
  now and there is currently no version available that compiles with any GHC newer than
  8.2. This fact posed a problem for me, because my own implementation works only on ghc
  8.4 and up. So I had to spend a bit of time and patch old version of its dependency
  [`fixed-vector`](https://github.com/lehins/fixed-vector/tree/add-semigroup-for-0.8.2)
  just so I can write some criterion benchmarks. The author Roman Leventov abandoned the
  project many years ago and the new maintainer Dominic Steinitz does not have enough
  interest to invest his time to actively improve the library. That said, I am sure he
  will not make you wait if you submit a PR with some improvements. At least that's my
  take on it.
* [`friday`](https://hackage.haskell.org/package/friday) package seems to be in a dying
  mode as well, since original author Raphael Javaux has moved on to other things
  and current maintainer [Thomas M. DuBuisson](https://github.com/TomMD) will probably [be
  happy to give up maintainership of the
  library](https://github.com/RaphaelJ/friday/pull/36#issuecomment-650442593) to a
  willing individual.

## Benchmarks

### Overview

The whole juice of this post is located in the
[haskell-benchmarks/canny](https://github.com/lehins/haskell-benchmarks/tree/598c464002db83e6fd7f58b66f8e1710194be455/canny)
repository so the code can be studied and results can be reproduced by any interested
party. There are some instructions in the README on how to run the benchmarks, reproduce
the results as well as some performance numbers generated by my old laptop.

In order to have objective numbers for the libraries we do need to concentrate on the
algorithm itself, therefore reading and writing of images have been abstracted away with
[`massiv-io`](https://github.com/lehins/massiv-io),
[`Color`](https://github.com/lehins/Color) and
[`JuicyPixels`](https://github.com/Twinside/Juicy.Pixels), none of which participate in
the benchmarks.


### Results


Without further ado here are some bar charts.

These are the runtimes for a pretty small 256x256 image of
[Lena](https://en.wikipedia.org/wiki/Lenna)

<!--
$ stack bench :bench-canny --ba '--output canny-small.html --match prefix Small --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv canny-small.html ../../lehins.github.io/assets/iframes/2020-07-10-canny-benchmarks/canny-small.html
-->

<iframe id="bench_iframe" frameborder="0" name="benchmarks" src="/assets/iframes/2020-07-10-canny-benchmarks/canny-small.html" style="overflow: hidden; width: 100%; height: 140px;" scrolling="auto">Here should be benchmarks</iframe>

<div style="text-align:center">
  <img src="/assets/images/canny-lena.png" alt="Lena - Canny Edge detection">
</div>


And this one is the same code applied to a slightly bigger 1920x1236 image of a cute
little frog:

<!--
stack bench :bench-canny --ba '--output canny-medium.html --match prefix Medium --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv canny-medium.html ../../lehins.github.io/assets/iframes/2020-07-10-canny-benchmarks/canny-medium.html
-->


<iframe id="bench_iframe" frameborder="0" name="benchmarks" src="/assets/iframes/2020-07-10-canny-benchmarks/canny-medium.html" style="overflow: hidden; width: 100%; height: 140px;" scrolling="auto">Here should be benchmarks</iframe>

<div style="text-align:center">
  <img src="/assets/images/canny-frog.png" alt="Frog - Canny Edge detection">
</div>



### Interpretation


`friday` has no means of parallelizing the computation, so it is not surprizing that it
performs like TGIF. Even if it did use all the available cores it wouldn't get anywhere
close to being competitive, just because it is lagging too far behind.

`yarr` claims being faster than `repa`, but it does not live up to its promise. Moreover,
considering that this package relies heavily on `fixed-vector`, which has experienced a
drastic change to its interface in the last two years, it would take good amount of work
in order to make `yarr` compatible with the newest version. It looks like yet another
array library has a pretty sad future ahead of it.

`repa` does lives up to its expectations of being fairly fast, but it is no longer the
fastest kid on the block. The biggest drawback of this library is lack of features. For
example, its stencil implementation is only capable of convolution, which prevents
combining computation of gradient, its magnitude and orientation in one iteration. Neither
`massiv` nor `accelerate` are susceptible to this problem.

`accelerate` does not do too well on small images, but that is the expected behavior and
in fact it is warned about by the authors. Reason for this is that the library does some
just-in-time compilation of the Haskell DSL into the target architecture, which on its own
takes time. This means that the computation you are trying to perform ought to be somewhat
heavy in order for the benefit to reveal itself. As far as I know this affect is even
further amplified for GPUs, because data has to be transferred over to/from GPU memory
before it can be operated on, but don't quote me on that. There are a few other
limitations that the library imposes on you, but that is not the point of this post, so I
am not going to expand on any of that.

`massiv` outperforms them all, at least in these benchmarks. I am usually not the kind of
person to be bragging about the stuff that I've done, but the results I've seen here speak
for themselves and I believe that `massiv` does deserve to be bragged about at least a
little. Its major strong points are:

* Pure Haskell implementation that relies only on GHC. Whenever LLVM backend is used GHC
  might produce even slightly faster code, but that is not a requirement.
* Automatic parallelization on all cores by the means of a specialized work stealing
  scheduler.
* When used correctly it will result in extremely efficient code. The API makes it harder
  for it to be used incorrectly through the power of Haskell's type system.
* Sophisticated and parallelizable stencil computation that works equally well for all
  dimensions, unlike `repa` or `accelerate` with two and three dimensions respectfully.
* Provides a fully featured interface with support for fusion and streams
* It is the only multi-dimensional array library with above properties that also has a
  full blown mutable interface
* Integration with pure Haskell libraries that read/write images and future integration
  with an image processing library.


### Individual steps


For larger images `massiv`, `repa` and `accelerate` are not too far from each other with
respect to performance, so I think it is worth to take a look at runtimes of individual
steps of the algorithm. This would allow me personally as a maintainer of `massiv` to see
where I can focus my attention in order to improve the library in the future.
<!--
stack bench :bench-canny-steps --ba '--output canny-steps.html --template /home/lehins/github/haskell-benchmarks/template.tpl' && mv canny-steps.html ../../lehins.github.io/assets/iframes/2020-07-10-canny-benchmarks/canny-steps.html
-->

<iframe id="bench_iframe" frameborder="0" name="benchmarks" src="/assets/iframes/2020-07-10-canny-benchmarks/canny-steps.html" style="overflow: hidden; width: 100%; height: 504px;" scrolling="auto">Here should be benchmarks</iframe>

Now let's look at a high level overview of individual steps of the algorithm and their
performance analysis.


#### Grayscale

Edge detection can't happen directly on the color image, so the first natural step is to
convert an sRGB encoded image into grayscale as luma (i.e. non-linear brightness of an
image). All of the libraries mistakenly call it luminance, but that is besides the
point. What matters is that the conversion happens in the exact same manner: `y = r *
0.299 + g * 0.587 + b * 0.114`. There are also arguments against using luma for some of
the steps in Canny, especually blur, but here I want to compare performance not accuracy,
so I went for consistency instead of correctness.

To my surprise, `accelerate` performed quite well at this rather simple step, much better
than the other two libraries. The only possible explanation I can have is that reliance on
LLVM allows it to automatically exploit SIMD instructions, which unfortunately isn't the
luxury that we have when compiling with GHC. Writing those instructions manually with GHC
primitives and LLVM backend is close to a nightmare and rarely yields any
benefit. Nevertheless it gives me ideas where I can spend more of my time in order to
improve performance even further.


#### Blur

At this step we remove unwanted noise by performing a gaussian blur, which involves
convolving the image twice. Once in horizontal and once more in vertical directions. This
separation into two convolutions is a very useful optimization and it is only
possible because gaussian kernel is separable. Both `repa` and `massiv` have nearly
identical performance, while `accelerate` is lagging behind for smaller images and would
likely catch up with larger size images.

#### Gradient

Besides computing the gradient with Sobel filter in this step we also need to compute its
magnitude and orientation. As mentioned earlier `repa` can't do it in a single step and
has to do three passes over an image, which has a negative impact on the
runtime. `accelerate` does combine all the substeps into a single stencil application,
nevertheless it doesn't help it enough to finish earlier than `massiv`.

#### Suppress

Runtime of non-maximum suppression pass does not seem to differ much between the
libraries. Similar to blur, all three seem to converge to equal runtime with larger sized
images.

#### Strong

Finding strong edges is essentially a filtering operation. Both `repa` and `accelerate`
try to run this step in parallel, which seems to only affect the performance in the
negative way. I found that it is better to rely on streams and do such
operations sequentially, similar to the way `vector` library does it. For this reason
`massiv` is quite a bit faster then the other guys.

#### Wildfire

Last step of Canny edge detection involves edge tracking by hysteresis, which sort of
looks like a wildfire, hence is the name I assume. This is an inherently imperative part
of the algorithm and relies heavily on mutation, which repa has no interface for and falls
back onto `vector` for the implementation. `accelerate` has no means of mutating arrays
either therefore it falls back on `repa` for the last stage, which again relies on
`vector` package. For that reason or another both of them are lagging behind `massiv`
quite a bit, which was design from the ground up to be a mutation friendly library.


## Conclusion


The only thing I can conclude here is that `massiv` is doing pretty good. As the author of
the library I mostly spent time on designing the interface and working on performance
optimizations. It is only recently with my work on `hip` that I finally got a chance to put
myself in the user's shoes at a scale larger than a few lines of code and so far I can say it's
been a pleasure. Both usability and performance make me think that the time I spent on it was
worth my while.

I'd like to take this opportunity and say thank you to everyone behind the work on
`vector`, `repa` and `accelerate`. All the resources produced over the years has taught me
a lot and served as inspiration for many design choices I made in `massiv`.
