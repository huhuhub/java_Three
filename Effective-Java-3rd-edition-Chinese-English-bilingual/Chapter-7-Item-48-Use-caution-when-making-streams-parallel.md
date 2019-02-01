## Chapter 7. Lambdas and Streams（λ表达式和流）

使流并行时要小心

### Item 48: Use caution when making streams parallel

Among mainstream languages, Java has always been at the forefront of providing facilities to ease the task of concurrent programming. When Java was released in 1996, it had built-in support for threads, with synchronization and wait/notify. Java 5 introduced the java.util.concurrent library, with concurrent collections and the executor framework. Java 7 introduced the fork-join package, a high-performance framework for parallel decomposition. Java 8 introduced streams, which can be parallelized with a single call to the parallel method. Writing concurrent programs in Java keeps getting easier, but writing concurrent programs that are correct and fast is as difficult as it ever was. Safety and liveness violations are a fact of life in concurrent programming, and parallel stream pipelines are no exception.

在主流语言中，Java始终处于提供便于并发编程任务的设施的最前沿。当Java于1996年发布时，它内置了对线程的支持，具有同步和等待/通知。 Java 5引入了java.util.concurrent库，包含并发集合和执行器框架。 Java 7引入了fork-join包，这是一个用于并行分解的高性能框架。 Java 8引入了流，可以通过对并行方法的单个调用来并行化。用Java编写并发程序变得越来越容易，但编写正确快速的并发程序就像以前一样困难。安全性和活性违规是并发编程中的事实，并行流管道也不例外。

Consider this program from Item 45:

考虑第45项中的这个程序：

```
// Stream-based program to generate the first 20 Mersenne primes

//基于流的程序生成前20个Mersenne素数

public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
    .filter(mersenne -> mersenne.isProbablePrime(50))
    .limit(20)
    .forEach(System.out::println);
}

static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

On my machine, this program immediately starts printing primes and takes 12.5 seconds to run to completion. Suppose I naively try to speed it up by adding a call to parallel() to the stream pipeline. What do you think will happen to its performance? Will it get a few percent faster? A few percent slower? Sadly, what happens is that it doesn’t print anything, but CPU usage spikes to 90 percent and stays there indefinitely (a liveness failure). The program might terminate eventually, but I was unwilling to find out; I stopped it forcibly after half an hour.

在我的机器上，该程序立即开始打印质数，并需要12.5秒才能完成运行。假设我天真地试图通过向流管道添加对parallel（）的调用来加速它。您认为它的表现会怎样？它会加快几个百分点吗？几个百分点慢？可悲的是，发生的事情是它没有打印任何东西，但是CPU使用率飙升至90％并且无限期地停留在那里（活跃度失败）。该计划最终可能会终止，但我不愿意发现;半小时后我强行停了下来。

What’s going on here? Simply put, the streams library has no idea how to parallelize this pipeline and the heuristics fail. Even under the best of circumstances, **parallelizing a pipeline is unlikely to increase its performance if the source is from Stream.iterate, or the intermediate operation limit is used.** This pipeline has to contend with both of these issues. Worse, the default parallelization strategy deals with the unpredictability of limit by assuming there’s no harm in processing a few extra elements and discarding any unneeded results. In this case, it takes roughly twice as long to find each Mersenne prime as it did to find the previous one. Thus, the cost of computing a single extra element is roughly equal to the cost of computing all previous elements combined, and this innocuous-looking pipeline brings the automatic parallelization algorithm to its knees. The moral of this story is simple: **Do not parallelize stream pipelines indiscriminately.** The performance consequences may be disastrous.

这里发生了什么？简而言之，流库不知道如何并行化此管道并且启发式失败。即使在最好的情况下，如果源来自Stream.iterate，或者使用中间操作限制，并行化管道也不太可能提高其性能。**这个管道必须应对这两个问题。更糟糕的是，默认的并行化策略通过假设处理一些额外元素并丢弃任何不需要的结果没有任何损害来处理限制的不可预测性。在这种情况下，找到每个Mersenne prime需要大约两倍的时间才能找到前一个。因此，计算单个额外元素的成本大致等于计算所有先前元素组合的成本，并且这种无害的管道使自动并行化算法瘫痪。这个故事的寓意很简单：**不要不加区别地对流管道进行并行化。**性能后果可能是灾难性的。

As a rule, **performance gains from parallelism are best on streams over ArrayList, HashMap, HashSet, and ConcurrentHashMap instances; arrays; int ranges; and long ranges.** What these data structures have in common is that they can all be accurately and cheaply split into subranges of any desired sizes, which makes it easy to divide work among parallel threads. The abstraction used by the streams library to perform this task is the spliterator, which is returned by the spliterator method on Stream and Iterable.

作为一项规则，来自并行性的**性能在ArrayList，HashMap，HashSet和ConcurrentHashMap实例上的流上最佳;阵列; int范围;这些数据结构的共同之处在于，它们都可以准确且廉价地分成任何所需大小的子范围，这使得在并行线程之间划分工作变得容易。流库用于执行此任务的抽象是spliterator，它由Stream和Iterable上的spliterator方法返回。

Another important factor that all of these data structures have in common is that they provide good-to-excellent locality of reference when processed sequentially: sequential element references are stored together in memory. The objects referred to by those references may not be close to one another in memory, which reduces locality-of-reference. Locality-of-reference turns out to be critically important for parallelizing bulk operations: without it, threads spend much of their time idle, waiting for data to be transferred from memory into the processor’s cache. The data structures with the best locality of reference are primitive arrays because the data itself is stored contiguously in memory.

所有这些数据结构的另一个重要因素是它们在顺序处理时提供了良好到极好的参考局部性：顺序元素引用一起存储在存储器中。这些引用所引用的对象在存储器中可能彼此不接近，这减少了引用的局部性。对于并行化批量操作而言，参考位置对于非常重要：如果没有它，线程会将大部分时间用在空闲状态，等待数据从内存传输到处理器的缓存中。具有最佳参考局部性的数据结构是原始阵列，因为数据本身连续存储在存储器中。

The nature of a stream pipeline’s terminal operation also affects the effectiveness of parallel execution. If a significant amount of work is done in the terminal operation compared to the overall work of the pipeline and that operation is inherently sequential, then parallelizing the pipeline will have limited effectiveness. The best terminal operations for parallelism are reductions, where all of the elements emerging from the pipeline are combined using one of Stream’s reduce methods, or prepackaged reductions such as min, max, count, and sum. The short-circuiting operations anyMatch, allMatch, and noneMatch are also amenable to parallelism. The operations performed by Stream’s collect method, which are known as mutable reductions, are not good candidates for parallelism because the overhead of combining collections is costly.

流管道终端操作的性质也会影响并行执行的有效性。如果与管道的整体工作相比在终端操作中完成了大量工作并且该操作本质上是顺序的，那么并行化管道将具有有限的有效性。并行性的最佳终端操作是减少，其中从管道中出现的所有元素使用Stream的reduce方法或预先打包的减少（例如min，max，count和sum）进行组合。 anyMatch，allMatch和noneMatch的短路操作也适用于并行操作。 Stream的collect方法执行的操作（称为可变约简）不是并行性的良好候选者，因为组合集合的开销很昂贵。

If you write your own Stream, Iterable, or Collection implementation and you want decent parallel performance, you must override the spliterator method and test the parallel performance of the resulting streams extensively. Writing high-quality spliterators is difficult and beyond the scope of this book.

如果您编写自己的Stream，Iterable或Collection实现并且希望获得良好的并行性能，则必须覆盖spliterator方法并广泛测试生成的流的并行性能。编写高质量的分裂器很困难，超出了本书的范围。

**Not only can parallelizing a stream lead to poor performance, including liveness failures; it can lead to incorrect results and unpredictable behavior** (safety failures). Safety failures may result from parallelizing a pipeline that uses mappers, filters, and other programmer-supplied function objects that fail to adhere to their specifications. The Stream specification places stringent requirements on these function objects. For example, the accumulator and combiner functions passed to Stream’s reduce operation must be associative, non-interfering, and stateless. If you violate these requirements (some of which are discussed in Item 46) but run your pipeline sequentially, it will likely yield correct results; if you parallelize it, it will likely fail, perhaps catastrophically. Along these lines, it’s worth noting that even if the parallelized Mersenne primes program had run to completion, it would not have printed the primes in the correct (ascending) order. To preserve the order displayed by the sequential version, you’d have to replace the forEach terminal operation with forEachOrdered, which is guaranteed to traverse parallel streams in encounter order.

**不仅可以并行化流导致性能不佳，包括活动失败;它可能导致不正确的结果和不可预测的行为**（安全故障）。使用映射器，过滤器和其他程序员提供的不符合其规范的功能对象的管道并行化可能会导致安全故障。 Stream规范对这些功能对象提出了严格的要求。例如，传递给Stream的reduce操作的累加器和组合器函数必须是关联的，非干扰的和无状态的。如果您违反了这些要求（其中一些在第46项中讨论过），但按顺序运行您的管道，则可能会产生正确的结果;如果你将它并行化，它可能会失败，也许是灾难性的。沿着这些思路，值得注意的是，即使并行化的Mersenne素数程序已经完成，它也不会以正确的（升序）顺序打印素数。要保留顺序版本显示的顺序，您必须使用forEachOrdered替换forEach终端操作，该操作保证以相遇顺序遍历并行流。

Even assuming that you’re using an efficiently splittable source stream, a parallelizable or cheap terminal operation, and non-interfering function objects, you won’t get a good speedup from parallelization unless the pipeline is doing enough real work to offset the costs associated with parallelism. As a very rough estimate, the number of elements in the stream times the number of lines of code executed per element should be at least a hundred thousand [Lea14].

即使假设您正在使用有效可拆分的源流，可并行化或廉价的终端操作以及非干扰功能对象，除非管道正在做足够的实际工作以抵消相关成本，否则您将无法从并行化获得良好的加速并行性。作为一个非常粗略的估计，流中元素的数量乘以每个元素执行的代码行数应该至少为十万[Lea14]。

It’s important to remember that parallelizing a stream is strictly a performance optimization. As is the case for any optimization, you must test the performance before and after the change to ensure that it is worth doing (Item 67). Ideally, you should perform the test in a realistic system setting. Normally, all parallel stream pipelines in a program run in a common fork-join pool. A single misbehaving pipeline can harm the performance of others in unrelated parts of the system.

重要的是要记住并行化流是严格的性能优化。与任何优化一样，您必须在更改之前和之后测试性能，以确保它值得做（第67项）。理想情况下，您应该在实际的系统设置中执行测试。通常，程序中的所有并行流管道都在公共fork-join池中运行。单个行为不当的管道可能会损害系统中不相关部分的其他行为。

If it sounds like the odds are stacked against you when parallelizing stream pipelines, it’s because they are. An acquaintance who maintains a multimillionline codebase that makes heavy use of streams found only a handful of places where parallel streams were effective. This does not mean that you should refrain from parallelizing streams. Under the right circumstances, it is possible to achieve near-linear speedup in the number of processor cores simply by adding a parallel call to a stream pipeline. Certain domains, such as machine learning and data processing, are particularly amenable to these speedups.

如果在并行化流水线时听起来像是堆积在你身上的几率，那是因为它们是。维持数百万行代码库的熟人大量使用流，只发现了平行流有效的少数几个地方。这并不意味着您应该避免并行化流。在适当的情况下，只需通过向流管道添加并行调用，就可以实现处理器内核数量的近线性加速。某些领域，例如机器学习和数据处理，特别适合这些加速。

As a simple example of a stream pipeline where parallelism is effective, consider this function for computing π(n), the number of primes less than or equal to n:

作为并行性有效的流管道的一个简单示例，请考虑此函数来计算π（n），素数小于或等于n：

```
// Prime-counting stream pipeline - benefits from parallelization
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
    .mapToObj(BigInteger::valueOf)
    .filter(i -> i.isProbablePrime(50))
    .count();
}
```

On my machine, it takes 31 seconds to compute π(108) using this function. Simply adding a parallel() call reduces the time to 9.2 seconds:

在我的机器上，使用此功能计算π（108）需要31秒。只需添加parallel（）调用即可将时间缩短为9.2秒：

```
// Prime-counting stream pipeline - parallel version
// Prime计数流管道 - 并行版本
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
    .parallel()
    .mapToObj(BigInteger::valueOf)
    .filter(i -> i.isProbablePrime(50))
    .count();
}
```

In other words, parallelizing the computation speeds it up by a factor of 3.7 on my quad-core machine. It’s worth noting that this is not how you’d compute π(n) for large values of n in practice. There are far more efficient algorithms, notably Lehmer’s formula.

换句话说，并行化计算可以在我的四核机器上将其加速3.7倍。值得注意的是，这并不是你在实践中如何计算大n值的π（n）。有更高效的算法，特别是Lehmer的公式。

If you are going to parallelize a stream of random numbers, start with a SplittableRandom instance rather than a ThreadLocalRandom (or the essentially obsolete Random). SplittableRandom is designed for precisely this use, and has the potential for linear speedup. ThreadLocalRandom is designed for use by a single thread, and will adapt itself to function as a parallel stream source, but won’t be as fast as SplittableRandom. Random synchronizes on every operation, so it will result in excessive, parallelism-killing contention.

如果要并行化随机数流，请从SplittableRandom实例开始，而不是ThreadLocalRandom（或基本上过时的Random）。 SplittableRandom专为此用途而设计，具有线性加速的潜力。 ThreadLocalRandom设计用于单个线程，并将自身适应作为并行流源，但不会像SplittableRandom一样快。随机同步每个操作，因此会导致过度的并行杀戮争用。

In summary, do not even attempt to parallelize a stream pipeline unless you have good reason to believe that it will preserve the correctness of the computation and increase its speed. The cost of inappropriately parallelizing a stream can be a program failure or performance disaster. If you believe that parallelism may be justified, ensure that your code remains correct when run in parallel, and do careful performance measurements under realistic conditions. If your code remains correct and these experiments bear out your suspicion of increased performance, then and only then parallelize the stream in production code.

总之，除非您有充分的理由相信它将保持计算的正确性并且不会尝试并行化流管道。