## Chapter 7. Lambdas and Streams（λ表达式和流）

优先选择流中无副作用的功能 

### Item 46: Prefer side-effect-free functions in streams

If you’re new to streams, it can be difficult to get the hang of them. Merely expressing your computation as a stream pipeline can be hard. When you succeed, your program will run, but you may realize little if any benefit. Streams isn’t just an API, it’s a paradigm based on functional programming. In order to obtain the expressiveness, speed, and in some cases parallelizability that streams have to offer, you have to adopt the paradigm as well as the API.

如果你是溪流新手，可能很难掌握它们。仅仅将您的计算表示为流管道可能很难。当你成功的时候，你的程序就会运行，但你可能几乎没有任何好处。 Streams不仅仅是一个API，它还是一个基于函数式编程的范例。为了获得流必须提供的表现力，速度和某些情况下的并行性，您必须采用范式以及API。

The most important part of the streams paradigm is to structure your computation as a sequence of transformations where the result of each stage is as close as possible to a pure function of the result of the previous stage. A pure function is one whose result depends only on its input: it does not depend on any mutable state, nor does it update any state. In order to achieve this, any function objects that you pass into stream operations, both intermediate and terminal, should be free of side-effects.

流范例中最重要的部分是将计算结构化为一系列转换，其中每个阶段的结果尽可能接近前一阶段结果的纯函数。纯函数的结果仅取决于其输入：它不依赖于任何可变状态，也不更新任何状态。为了实现这一点，您传递给流操作的任何函数对象（中间和终端）都应该没有副作用。

Occasionally, you may see streams code that looks like this snippet, which builds a frequency table of the words in a text file:

有时，您可能会看到类似于此代码段的流代码，它会在文本文件中构建单词的频率表：

```
// Uses the streams API but not the paradigm(范例)--Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

What’s wrong with this code? After all, it uses streams, lambdas, and method references, and gets the right answer. Simply put, it’s not streams code at all; it’s iterative code masquerading as streams code. It derives no benefits from the streams API, and it’s (a bit) longer, harder to read, and less maintainable than the corresponding iterative code. The problem stems from the fact that this code is doing all its work in a terminal forEach operation, using a lambda that mutates external state (the frequency table). A forEach operation that does anything more than present the result of the computation performed by a stream is a “bad smell in code,” as is a lambda that mutates state. So how should this code look?

这段代码出了什么问题？毕竟，它使用流，lambdas和方法引用，并得到正确的答案。简单地说，它根本不是流代码;它的迭代代码伪装成流代码。它没有从流API中获益，并且它比相应的迭代代码更长，更难以阅读，并且维护更少。问题源于这样一个事实：这个代码在一个终端forEach操作中完成所有工作，使用一个变异外部状态的lambda（频率表）。执行除了呈现流执行的计算结果之外的任何操作的forEach操作都是“代码中的难闻气味”，因为是一个变异状态的lambda。那么这段代码应该怎么样？

```
// Proper use of streams to initialize a frequency table
// 正确使用流来初始化频率表
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words.collect(groupingBy(String::toLowerCase, counting()));
}
```

This snippet does the same thing as the previous one but makes proper use of the streams API. It’s shorter and clearer. So why would anyone write it the other way? Because it uses tools they’re already familiar with. Java programmers know how to use for-each loops, and the forEach terminal operation is similar. But the forEach operation is among the least powerful of the terminal operations and the least stream-friendly. It’s explicitly iterative, and hence not amenable to parallelization. **The forEach operation should be used only to report the result of a stream computation, not to perform the computation.** Occasionally, it makes sense to use forEach for some other purpose, such as adding the results of a stream computation to a preexisting collection.

此代码段与前一代码相同，但正确使用了流API。它更短更清晰。那么为什么有人会用另一种方式写呢？因为它使用了他们已经熟悉的工具。 Java程序员知道如何使用for-each循环，而forEach终端操作是类似的。但forEach操作是终端操作中最不强大的操作之一，也是最不友好的流操作。它是明确的迭代，因此不适合并行化。 ** forEach操作应该仅用于报告流计算的结果，而不是用于执行计算。**偶尔，将forEach用于其他目的是有意义的，例如将流计算的结果添加到a预先存在的收藏。

The improved code uses a collector, which is a new concept that you have to learn in order to use streams. The Collectors API is intimidating: it has thirty-nine methods, some of which have as many as five type parameters. The good news is that you can derive most of the benefit from this API without delving into its full complexity. For starters, you can ignore the Collector interface and think of a collector as an opaque object that encapsulates a reduction strategy. In this context, reduction means combining the elements of a stream into a single object. The object produced by a collector is typically a collection (which accounts for the name collector).

改进的代码使用了一个收集器，这是一个新概念，您必须学习才能使用流。 Collectors API令人生畏：它有三十九种方法，其中一些方法有多达五种类型参数。好消息是，您可以从这个API中获得大部分好处，而无需深入研究其完整的复杂性。对于初学者，您可以忽略Collector接口，并将收集器视为封装缩减策略的不透明对象。在这种情况下，缩减意味着将流的元素组合成单个对象。收集器生成的对象通常是一个集合（它代表名称收集器）。

The collectors for gathering the elements of a stream into a true Collection are straightforward. There are three such collectors: toList(), toSet(), and toCollection(collectionFactory). They return, respectively, a set, a list, and a programmer-specified collection type. Armed with this knowledge, we can write a stream pipeline to extract a top-ten list from our frequency table.

用于将流的元素收集到真正的集合中的收集器是直截了当的。有三个这样的收集器：toList（），toSet（）和toCollection（collectionFactory）。它们分别返回一个集合，一个列表和一个程序员指定的集合类型。有了这些知识，我们可以编写一个流管道来从频率表中提取前十个列表。

```
// Pipeline to get a top-ten list of words from a frequency table
// 管道从频率表中获取前十个单词列表
List<String> topTen = freq.keySet().stream()
.sorted(comparing(freq::get).reversed())
.limit(10)
.collect(toList());
```

Note that we haven’t qualified the toList method with its class, Collectors. **It is customary and wise to statically import all members of Collectors because it makes stream pipelines more readable.**

请注意，我们没有使用其类Collectors限定toList方法。 **静态导入收集器的所有成员是习惯和明智的，因为它使流管道更具可读性。**

The only tricky part of this code is the comparator that we pass to sorted, comparing(freq::get).reversed(). The comparing method is a comparator construction method (Item 14) that takes a key extraction function. The function takes a word, and the “extraction” is actually a table lookup: the bound method reference freq::get looks up the word in the frequency table and returns the number of times the word appears in the file. Finally, we call reversed on the comparator, so we’re sorting the words from most frequent to least frequent. Then it’s a simple matter to limit the stream to ten words and collect them into a list.

这段代码中唯一棘手的部分是我们传递给sorted，compare（freq :: get）.reversed（）的比较器。比较方法是采用密钥提取功能的比较器构造方法（第14项）。该函数接受一个单词，“extract”实际上是一个表查找：绑定方法引用freq :: get在频率表中查找单词并返回单词在文件中出现的次数。最后，我们在比较器上调用reverse，因此我们将单词从最频繁到最不频繁地排序。然后将流限制为十个单词并将它们收集到一个列表中是一件简单的事情。

The previous code snippets use Scanner’s stream method to get a stream over the scanner. This method was added in Java 9. If you’re using an earlier release, you can translate the scanner, which implements Iterator, into a stream using an adapter similar to the one in Item 47 (streamOf(Iterable<E>)).

之前的代码片段使用Scanner的流方法在扫描程序上获取流。在Java 9中添加了此方法。如果您使用的是早期版本，则可以使用类似于第47项（streamOf（Iterable <E>））的适配器将实现Iterator的扫描程序转换为流。

So what about the other thirty-six methods in Collectors? Most of them exist to let you collect streams into maps, which is far more complicated than collecting them into true collections. Each stream element is associated with a key and a value, and multiple stream elements can be associated with the same key.

那么收藏家的其他36种方法呢？它们中的大多数存在是为了让您将流收集到地图中，这比将它们收集到真实集合中要复杂得多。每个流元素与键和值相关联，并且多个流元素可以与相同的键相关联。

The simplest map collector is toMap(keyMapper, valueMapper), which takes two functions, one of which maps a stream element to a key, the other, to a value. We used this collector in our fromString implementation in Item 34 to make a map from the string form of an enum to the enum itself:

最简单的映射收集器是toMap（keyMapper，valueMapper），它接受两个函数，其中一个函数将一个流元素映射到一个键，另一个函数映射到一个值。我们在第34项的fromString实现中使用了这个收集器来创建从枚举的字符串形式到枚举本身的映射：

```
// Using a toMap collector to make a map from string to enum
// 使用toMap收集器创建从字符串到枚举的映射
private static final Map<String, Operation> stringToEnum =Stream.of(values()).collect(toMap(Object::toString, e -> e));
```

This simple form of toMap is perfect if each element in the stream maps to a unique key. If multiple stream elements map to the same key, the pipeline will terminate with an IllegalStateException.
如果流中的每个元素都映射到唯一键，则这种简单的toMap形式是完美的。如果多个流元素映射到同一个键，则管道将以IllegalStateException终止。

The more complicated forms of toMap, as well as the groupingBy method, give you various ways to provide strategies for dealing with such collisions. One way is to provide the toMap method with a merge function in addition to its key and value mappers. The merge function is a BinaryOperator<V>, where V is the value type of the map. Any additional values associated with a key are combined with the existing value using the merge function, so, for example, if the merge function is multiplication, you end up with a value that is the product of all the values associated with the key by the value mapper.

更复杂的toMap形式以及groupingBy方法为您提供了各种方法来提供处理此类冲突的策略。一种方法是除了键和值映射器之外，还为toMap方法提供合并函数。合并函数是BinaryOperator <V>，其中V是映射的值类型。使用合并函数将与键关​​联的任何其他值与现有值组合，因此，例如，如果合并函数是乘法，则最终得到的值是与键关联的所有值的乘积。价值映射器。

The three-argument form of toMap is also useful to make a map from a key to a chosen element associated with that key. For example, suppose we have a stream of record albums by various artists, and we want a map from recording artist to best-selling album. This collector will do the job.

toMap的三参数形式对于创建从键到与该键关联的所选元素的映射也很有用。例如，假设我们有各种艺术家的唱片专辑流，我们想要一张从录音艺术家到最畅销专辑的地图。这个收藏家将完成这项工作。

```
// Collector to generate a map from key to chosen element for key
//收集器生成从键到选定元素的键的映射
Map<Artist, Album> topHits = albums.collect(
        toMap(Album::artist, a->a, maxBy(comparing(Album::sales)
    )
));
```

Note that the comparator uses the static factory method maxBy, which is statically imported from BinaryOperator. This method converts a Comparator<T> into a BinaryOperator<T> that computes the maximum implied by the specified comparator. In this case, the comparator is returned by the comparator construction method comparing, which takes the key extractor function Album::sales. This may seem a bit convoluted, but the code reads nicely. Loosely speaking, it says, “convert the stream of albums to a map, mapping each artist to the album that has the best album by sales.” This is surprisingly close to the problem statement.

请注意，比较器使用静态工厂方法maxBy，它是从BinaryOperator静态导入的。此方法将Comparator <T>转换为BinaryOperator <T>，用于计算指定比较器隐含的最大值。在这种情况下，比较器由比较器构造方法比较返回，它采用密钥提取器功能Album :: sales。这可能看起来有点复杂，但代码读得很好。简而言之，它说，“将专辑流转换为地图，将每位艺术家映射到销售量最佳专辑的专辑。”这令人惊讶地接近问题陈述。

Another use of the three-argument form of toMap is to produce a collector that imposes a last-write-wins policy when there are collisions. For many streams, the results will be nondeterministic, but if all the values that may be associated with a key by the mapping functions are identical, or if they are all acceptable, this collector’s s behavior may be just what you want:

toMap的三参数形式的另一个用途是产生一个收集器，当发生冲突时强制执行last-write-wins策略。对于许多流，结果将是不确定的，但如果映射函数可能与键关联的所有值都相同，或者它们都是可接受的，则此收集器的行为可能正是您想要的：

```
// Collector to impose last-write-wins policy
// 收集者实施最后写入 - 获胜政策
toMap(keyMapper, valueMapper, (v1, v2) -> v2)
```

The third and final version of toMap takes a fourth argument, which is a map factory, for use when you want to specify a particular map implementation such as an EnumMap or a TreeMap.

toMap的第三个也是最后一个版本采用第四个参数，即一个地图工厂，用于指定特定的地图实现，例如EnumMap或TreeMap。

There are also variant forms of the first three versions of toMap, named toConcurrentMap, that run efficiently in parallel and produce ConcurrentHashMap instances.

toMap的前三个版本也有变体形式，名为toConcurrentMap，它们并行高效运行并生成ConcurrentHashMap实例。

In addition to the toMap method, the Collectors API provides the groupingBy method, which returns collectors to produce maps that group elements into categories based on a classifier function. The classifier function takes an element and returns the category into which it falls. This category serves as the element’s map key. The simplest version of the groupingBy method takes only a classifier and returns a map whose values are lists of all the elements in each category. This is the collector that we used in the Anagram program in Item 45 to generate a map from alphabetized word to a list of the words sharing the alphabetization:

除了toMap方法之外，Collectors API还提供了groupingBy方法，该方法返回收集器以生成基于分类器函数将元素分组到类别中的映射。分类器函数接受一个元素并返回它所属的类别。此类别用作元素的地图键。 groupingBy方法的最简单版本仅采用分类器并返回一个映射，其值是每个类别中所有元素的列表。这是我们在第45项中的Anagram程序中使用的收集器，用于生成从按字母顺序排列的单词到共享字母顺序的单词列表的地图：

```
words.collect(groupingBy(word -> alphabetize(word)))
```

If you want groupingBy to return a collector that produces a map with values other than lists, you can specify a downstream collector in addition to a classifier. A downstream collector produces a value from a stream containing all the elements in a category. The simplest use of this parameter is to pass toSet(), which results in a map whose values are sets of elements rather than lists.

如果希望groupingBy返回一个生成带有除列表之外的值的映射的收集器，则除了分类器之外，还可以指定下游收集器。下游收集器从包含类别中所有元素的流生成值。此参数的最简单用法是传递toSet（），这将生成一个映射，其值是元素集而不是列表。

Alternatively, you can pass toCollection(collectionFactory), which lets you create the collections into which each category of elements is placed. This gives you the flexibility to choose any collection type you want. Another simple use of the two-argument form of groupingBy is to pass counting() as the downstream collector. This results in a map that associates each category with the number of elements in the category, rather than a collection containing the elements. That’s what you saw in the frequency table example at the beginning of this item:

或者，您可以传递toCollection（collectionFactory），它允许您创建放置每个元素类别的集合。这使您可以灵活地选择所需的任何集合类型。另一种简单使用groupingBy的双参数形式的方法是将counting（）作为下游收集器传递。这会生成一个映射，该映射将每个类别与类别中的元素数相关联，而不是包含元素的集合。这就是您在本项目开头的频率表示例中看到的内容：

```
Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));
```

The third version of groupingBy lets you specify a map factory in addition to a downstream collector. Note that this method violates the standard telescoping argument list pattern: the mapFactory parameter precedes, rather than follows, the downStream parameter. This version of groupingBy gives you control over the containing map as well as the contained collections, so, for example, you can specify a collector that returns a TreeMap whose values are TreeSets.

groupingBy的第三个版本允许您指定除下游收集器之外的地图工厂。请注意，此方法违反了标准的telescoping参数列表模式：mapFactory参数位于downStream参数之前，而不是之后。此版本的groupingBy使您可以控制包含的映射以及包含的集合，因此，例如，您可以指定一个收集器，该收集器返回值为TreeSet的TreeMap。

The groupingByConcurrent method provides variants of all three overloadings of groupingBy. These variants run efficiently in parallel and produce ConcurrentHashMap instances. There is also a rarely used relative of groupingBy called partitioningBy. In lieu of a classifier method, it takes a predicate and returns a map whose key is a Boolean. There are two overloadings of this method, one of which takes a downstream collector in addition to a predicate.

groupingByConcurrent方法提供了groupingBy的所有三个重载的变体。这些变体并行高效运行并生成ConcurrentHashMap实例。还有一个很少使用的grouping的亲戚叫做partitioningBy。代替分类器方法，它接受谓词并返回其键为布尔值的映射。此方法有两个重载，其中一个除谓词之外还包含下游收集器。

The collectors returned by the counting method are intended only for use as downstream collectors. The same functionality is available directly on Stream, via the count method, so **there is never a reason to say collect(counting()).** There are fifteen more Collectors methods with this property. They include the nine methods whose names begin with summing, averaging, and summarizing (whose functionality is available on the corresponding primitive stream types). They also include all overloadings of the reducing method, and the filtering, mapping, flatMapping, and collectingAndThen methods. Most programmers can safely ignore the majority of these methods. From a design perspective, these collectors represent an attempt to partially duplicate the functionality of streams in collectors so that downstream collectors can act as “ministreams.”

通过计数方法返回的收集器仅用作下游收集器。可以通过count方法直接在Stream上使用相同的功能，因此**没有理由说使用collect（counting（））。**此属性还有十五个Collectors方法。它们包括九个方法，其名称以求和，平均和汇总开头（其功能在相应的原始流类型上可用）。它们还包括reduce方法的所有重载，以及filter，mapping，flatMapping和collectingAndThen方法。大多数程序员可以安全地忽略大多数这些方法。从设计角度来看，这些收集器代表了尝试在收集器中部分复制流的功能，以便下游收集器可以充当“迷你流”。

There are three Collectors methods we have yet to mention. Though they are in Collectors, they don’t involve collections. The first two are minBy and maxBy, which take a comparator and return the minimum or maximum element in the stream as determined by the comparator. They are minor generalizations of the min and max methods in the Stream interface and are the collector analogues of the binary operators returned by the like-named methods in BinaryOperator. Recall that we used BinaryOperator.maxBy in our best-selling album example.

我们还有三种收藏家方法尚未提及。虽然他们在收藏家，但他们不涉及收藏。前两个是minBy和maxBy，它们取比较器并返回由比较器确定的流中的最小或最大元素。它们是Stream接口中min和max方法的次要推广，是BinaryOperator中类似命名方法返回的二元运算符的收集器类似物。回想一下，我们在最畅销的专辑中使用了BinaryOperator.maxBy。

The final Collectors method is joining, which operates only on streams of CharSequence instances such as strings. In its parameterless form, it returns a collector that simply concatenates the elements. Its one argument form takes a single CharSequence parameter named delimiter and returns a collector that joins the stream elements, inserting the delimiter between adjacent elements. If you pass in a comma as the delimiter, the collector returns a comma-separated values string (but beware that the string will be ambiguous if any of the elements in the stream contain commas). The three argument form takes a prefix and suffix in addition to the delimiter. The resulting collector generates strings like the ones that you get when you print a collection, for example [came, saw, conquered].

最后的Collectors方法是join，它只对CharSequence实例的流进行操作，例如字符串。在其无参数形式中，它返回一个简单地连接元素的收集器。它的一个参数形式采用名为delimiter的单个CharSequence参数，并返回一个连接流元素的收集器，在相邻元素之间插入分隔符。如果传入逗号作为分隔符，则收集器将返回逗号分隔值字符串（但请注意，如果流中的任何元素包含逗号，则字符串将不明确）。除了分隔符之外，三个参数形式还带有前缀和后缀。生成的收集器会生成类似于打印集合时获得的字符串，例如[来，看到，征服]。

In summary, the essence of programming stream pipelines is side-effect-free function objects. This applies to all of the many function objects passed to streams and related objects. The terminal operation forEach should only be used to report the result of a computation performed by a stream, not to perform the computation. In order to use streams properly, you have to know about collectors. The most important collector factories are toList, toSet, toMap, groupingBy, and joining.

总之，编程流管道的本质是无副作用的功能对象。这适用于传递给流和相关对象的所有许多函数对象。终端操作forEach仅应用于报告流执行的计算结果，而不是用于执行计算。为了正确使用流，您必须了解收集器。最重要的收集器工厂是toList，toSet，toMap，groupingBy和join。
