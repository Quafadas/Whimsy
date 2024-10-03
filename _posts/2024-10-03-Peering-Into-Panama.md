---
title: Peering into Panama
---

I've been recently spending time on [Vecxt](https://github.com/quafadas/vecxt), which is a cross platfrom linear algebra library that I write for my own curiosity. The core promise of vecxt, is that it produces best-on-platform performance. On the JVM, the future of that _should_ look like (at the time of writing incubating) [project panama](https://openjdk.org/projects/panama/). 

Here are my notes.

# TL:DR

Desipte being on something like the 8th incubator, there is definitely still an "incubating" feel to it. Key takeaways:

- The JVM does a good job already for "simple" algorithms
- A poorly implemented SIMD algorithm _will_ tank your performance, in some cases by up to 200x. 
- A _well_ implemented SIMD algorthim will tank your performance... on the wrong platform
- Working with project panama right now is essentially benchmark driven development. You _need_ [jmh](https://github.com/openjdk/jmh). 

# Experiments

## Simple cases

### Sum
More or less the first thing I'd expect anyone to do with the Vector API, is sum a list of doubles. Even with something this simple, there are a few ways to [skin this cat](https://github.com/Quafadas/vecxt/blob/main/benchmark/src/sum.scala). For this trivial algorthim, there is mental overhead in the reading / writing. 

```scala

  extension (vec: Array[Double])

    inline def sum: Double =
      var i: Int = 0
      val sp = DoubleVector.SPECIES_PREFERRED
      var acc = DoubleVector.zero(sp)

      val l = sp.length()

      while i < sp.loopBound(vec.length) do
        acc = acc.add(DoubleVector.fromArray(sp, vec, i))
        i += l
      end while
      var temp = acc.reduceLanes(VectorOperators.ADD)
      // var temp = 0.0
      while i < vec.length do
        temp += vec(i)
        i += 1
      end while
      temp
    end sum


    inline def sum2 =
      var sum: Double = 0.0
      var i: Int = 0
      val sp = Matrix.Matrix.doubleSpecies
      val l = sp.length()

      while i < sp.loopBound(vec.length) do
        sum = sum + DoubleVector.fromArray(sp, vec, i).reduceLanes(VectorOperators.ADD)
        i += l
      end while
      while i < vec.length do
        sum += vec(i)
        i += 1
      end while
      sum
    end sum2

    inline def sum3 =
      var sum: Double = 0.0
      var i: Int = 0
      while i < vec.length do
        sum = sum + vec(i)
        i = i + 1
      end while
      sum
    end sum3
```

The first implementation is the one we used, and the results (sum_vec) are, at first glance quite compelling vs the JVM loop (sum_loop)! 

![screencap](/docs/assets/sum_bench.png)

The first array has size 3, which means it should not be vectorised. That the results and errors bars are similar, should give us a control that the measurements stand some chance of being meaningful. 

As the array size increases, we appear to get better results. In fact, it looks like we can hit almost 4x for large arrays, which (coincidentally?) is the number of lanes for the preferred double species on the hardware, that the benchmark ran on. Perhaps, this shows us the promise of panama and SIMD. 

### Booleans

The benefits of the VectorAPI are particulaly profound, for where you have many lanes. As the Boolean Species has a very wide "shape", they are be processed 64 (!) at a time in a single CPU cycle. If you happen to be working through a large number of logical operations, Panama is an easy, and no brains winner! I benchmarked it at about a 10x speedup vs the "obvious" implementation. In this case I believe memory access bounds the performance gain. 

Still, a whole order of magnitude speed up for little brainspace is... attractive. 

## Gotchas

Panama does not guarantee hardware acceleration. It also won't tell you when it is not hardware accelerating and the implications can be ... bad. It fulfils the promise of doing the calculation (correctly), but the fallback mechanism can be ... sluggish. And sluglike.

### Scala

There is somerthing about scala itself (at the moment), which seems to trip up the hardware acceleration under hard to detect cuircumstances. 

```
Benchmark                     (len)   Mode  Cnt       Score        Error  Units
IndexBenchmark.index_loop     10000  thrpt    3  476661.430 ± 333415.679  ops/s
IndexBenchmark.index_vec      10000  thrpt    3  821343.888 ±  22086.257  ops/s
IndexBenchmark.index_vec_bad  10000  thrpt    3   31524.934 ±  59643.768  ops/s
```

In the "vec bad" case, I did little other than declare the species (of the same shape) like this;

```scala
val spi = IntVector.SPECIES_PREFERRED
val sp = VectorSpecies.of(java.lang.Integer.TYPE, spi.vectorShape())
```

Instead of only this. 

```scala 
val spi = IntVector.SPECIES_PREFERRED
```

Yet the performance impact was catastrophic. In java, one must declare 

```static final VectorSpecies<Integer>  sp = VectorSpecies.of(java.lang.Integer.TYPE, spi.vectorShape())```

So that the compiler knows... something... about it which allows the hardware optimisations to kick in. The extact same code written in java, does hardware accelerate, which is great. It seems `val` is apparently not _quite_ the same as static final - as one observes through the order of magniture performance hit. Here be dragons. Benchmark carefully.

### Integers

So let's say we're generating indexes.

```scala
val arr = Array.ofDim[Int](lenInt)
val sp = IntVector.SPECIES_PREFERRED
val l = sp.length()
var i = 0

while i < sp.loopBound(lenInt) do
  val v : IntVector = IntVector.broadcast(sp,i).addIndex(1) 
  v.intoArray(arr, i)
  i += l
end

```
This code will run at least on a par with the obvious "looped" implementation. Now, we want to multiply 

```java
IntVector iVec = indexes.mul(2);
```
No problem - speedy! But this?

```java
IntVector iVec = indexes.div(2);
```

WTF?
```
Benchmark                      (implementation)  (len)   Mode  Cnt       Score   Error  Units
IndexJBenchmark.index_vec                  java  10000  thrpt       328433.749          ops/s
IndexJBenchmark.index_vec_div              java  10000  thrpt        24136.053          ops/s
IndexJBenchmark.index_vec_mul              java  10000  thrpt       343136.874          ops/s
```

The answer it turns out, is both reasonable and in hindisght obvious (I emailed the project panama list). It turns out, that there actually isn't a vector hardware instruction (on x86) for integral values (!). This is surprising, but really quite fundamental. Java cannot magic new hardware instructions. **

What it can do, is fallback to it's slow (10x) path. 

Currently, knowing whether you're hitting the hardware accelerated path or not basically... means using JMH or reading the compiled artefacts. There are no warnings or diagnostics. Make no assumptions, benchmark carefully.

** This raises the question over whether the method _should_ be in the API? The answer is above my paygrade, but considering the implied lowest common hardware bound across all operators and the dimensional explosion of documentation, there isn't a good answer. Perhaps a consistent API surface is the best solution.Hard problem. 

## Complex cases

As soon as one moves beyond trivial algorithms, the "free lunch" drops off quite quickly. I tried (and failed) to achieve performance benefits on two "more complex" algorithms. In both cases I could write a "correct" implementation. Both cases bled 10x (i.e. 10x slower) than the simple loop. 

### SIMD matrix slicing


### Prefix Sum

Because this is a hobby project, I can do silly things like try to optimise the cumulative sum of an array. It's a cool case, because the algorithm is trivial to write in a simple loop, and explain. 

```scala 
extension (vec: Array[Double])
  inline def cumsum: Unit =
    var i = 1
    while i < vec.length do
      vec(i) = vec(i - 1) + vec(i)
      i = i + 1
    end while
  end cumsum

```

And rather resistant to optimisation or parralisation. There are parrallel algorthims ...

https://en.wikipedia.org/wiki/Prefix_sum#:~:text=In%20computer%20science,%20the%20prefix%20sum,#:~:text=In%20computer%20science,%20the%20prefix%20sum,

https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda#:~:text=A%20simple%20and%20common%20parallel%20algorithm#:~:text=A%20simple%20and%20common%20parallel%20algorithm

... but they are complex. Here's my implentation odf the prefix sum. 

https://github.com/Quafadas/vecxt/blob/93df12f206308751ef7dc5318e97a1d64d0ac0fc/vecxt/jvm/src/arrays.scala#L303

It appears to pass the unit tests, and - Surprise! It performs about 100x worse than a simple while loop. 

It only took most of a day to find that out... but self evidently, it isn't worth including. 