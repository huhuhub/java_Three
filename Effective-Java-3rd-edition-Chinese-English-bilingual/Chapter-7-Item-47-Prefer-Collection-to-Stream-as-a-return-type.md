## Chapter 7. Lambdas and Streams（λ表达式和流）

首选Collection to Stream作为返回类型
### Item 47: Prefer Collection to Stream as a return type

Many methods return sequences of elements. Prior to Java 8, the obvious return types for such methods were the collection interfaces Collection, Set, and List; Iterable; and the array types. Usually, it was easy to decide which of these types to return. The norm was a collection interface. If the method existed solely to enable for-each loops or the returned sequence couldn’t be made to implement some Collection method (typically, contains(Object)), the Iterable interface was used. If the returned elements were primitive values or there were stringent performance requirements, arrays were used. In Java 8, streams were added to the platform, substantially complicating the task of choosing the appropriate return type for a sequence-returning method.

许多方法返回元素序列。在Java 8之前，这些方法的明显返回类型是集合接口Collection，Set和List;可迭代;和数组类型。通常，很容易决定返回哪些类型。规范是一个集合界面。如果该方法仅用于启用for-each循环或返回的序列无法实现某些Collection方法（通常为contains（Object）），则使用Iterable接口。如果返回的元素是原始值或者存在严格的性能要求，则使用数组。在Java 8中，流被添加到平台中，这使得为序列返回方法选择适当的返回类型的任务变得非常复杂。

You may hear it said that streams are now the obvious choice to return a sequence of elements, but as discussed in Item 45, streams do not make iteration obsolete: writing good code requires combining streams and iteration judiciously. If an API returns only a stream and some users want to iterate over the returned sequence with a for-each loop, those users will be justifiably upset. It is especially frustrating because the Stream interface contains the sole abstract method in the Iterable interface, and Stream’s specification for this method is compatible with Iterable’s. The only thing preventing programmers from using a for-each loop to iterate over a stream is Stream’s failure to extend Iterable.

您可能听说过，流现在是返回一系列元素的明显选择，但如第45项所述，流不会使迭代过时：编写好的代码需要明智地组合流和迭代。如果API只返回一个流，而某些用户想要使用for-each循环迭代返回的序列，那么这些用户将有理由感到不安。特别令人沮丧的是，Stream接口包含Iterable接口中唯一的抽象方法，Stream的此方法规范与Iterable兼容。阻止程序员使用for-each循环迭代流的唯一因素是Stream无法扩展Iterable。

Sadly, there is no good workaround for this problem. At first glance, it might appear that passing a method reference to Stream’s iterator method would work. The resulting code is perhaps a bit noisy and opaque, but not unreasonable:

可悲的是，这个问题没有好的解决方法。乍一看，似乎可以将方法引用传递给Stream的迭代器方法。结果代码可能有点嘈杂和不透明，但并非不合理：

```
// Won't compile, due to limitations on Java's type inference
//由于Java类型推断的限制，无法编译
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // Process the process
}
```

Unfortunately, if you attempt to compile this code, you’ll get an error message:

不幸的是，如果您尝试编译此代码，您将收到一条错误消息：

```
Test.java:6: error: method reference not expected here
//错误：此处不期望方法引用
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
^
```

In order to make the code compile, you have to cast the method reference to an appropriately parameterized Iterable:

为了使代码编译，您必须将方法引用强制转换为适当参数化的Iterable：

```
// Hideous workaround to iterate over a stream
// 用于迭代流的隐藏变通方法
for (ProcessHandle ph : (Iterable<ProcessHandle>)
    ProcessHandle.allProcesses()::iterator)
```

This client code works, but it is too noisy and opaque to use in practice. A better workaround is to use an adapter method. The JDK does not provide such a method, but it’s easy to write one, using the same technique used in-line in the snippets above. Note that no cast is necessary in the adapter method because Java’s type inference works properly in this context:

此客户端代码有效，但在实践中使用它太嘈杂和不透明。更好的解决方法是使用适配器方法。 JDK没有提供这样的方法，但是使用上面的代码片段中使用的相同技术，可以很容易地编写一个方法。请注意，在适配器方法中不需要强制转换，因为Java的类型推断在此上下文中正常工作：

```
// Adapter from Stream<E> to Iterable<E>
// 适配器从Stream <E>到Iterable <E>
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}
```

With this adapter, you can iterate over any stream with a for-each statement:

使用此适配器，您可以使用for-each语句迭代任何流：

```
for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
    // Process the process
}
```

Note that the stream versions of the Anagrams program in Item 34 use the Files.lines method to read the dictionary, while the iterative version uses a scanner. The Files.lines method is superior to a scanner, which silently swallows any exceptions encountered while reading the file. Ideally, we would have used Files.lines in the iterative version too. This is the sort of compromise that programmers will make if an API provides only stream access to a sequence and they want to iterate over the sequence with a for-each statement.

请注意，第34项中的Anagrams程序的流版本使用Files.lines方法读取字典，而迭代版本使用扫描程序。 Files.lines方法优于扫描程序，它可以在读取文件时静默吞下任何异常。理想情况下，我们也会在迭代版本中使用Files.lines。如果API仅提供对序列的流访问并且他们希望使用for-each语句迭代序列，那么程序员将会做出这种妥协。

Conversely, a programmer who wants to process a sequence using a stream pipeline will be justifiably upset by an API that provides only an Iterable. Again the JDK does not provide an adapter, but it’s easy enough to write one:

相反，想要使用流管道处理序列的程序员将完全被仅提供Iterable的API所打乱。 JDK再次没有提供适配器，但写一个很容易：

```
// Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

If you’re writing a method that returns a sequence of objects and you know that it will only be used in a stream pipeline, then of course you should feel free to return a stream. Similarly, a method returning a sequence that will only be used for iteration should return an Iterable. But if you’re writing a public API that returns a sequence, you should provide for users who want to write stream pipelines as well as those who want to write for-each statements, unless you have a good reason to believe that most of your users will want to use the same mechanism.

如果您正在编写一个返回一系列对象的方法，并且您知道它只会在流管道中使用，那么您当然可以随意返回一个流。类似地，返回仅用于迭代的序列的方法应返回Iterable。但是，如果您正在编写一个返回序列的公共API，那么您应该为想要编写流管道的用户以及想要为每个语句编写的用户提供，除非您有充分的理由相信大多数用户希望使用相同的机制。

The Collection interface is a subtype of Iterable and has a stream method, so it provides for both iteration and stream access. Therefore, **Collection or an appropriate subtype is generally the best return type for a public, sequence-returning method.** Arrays also provide for easy iteration and stream access with the Arrays.asList and Stream.of methods. If the sequence you’re returning is small enough to fit easily in memory, you’re probably best off returning one of the standard collection implementations, such as ArrayList or HashSet. But **do not store a large sequence in memory just to return it as a collection.** 

Collection接口是Iterable的子类型，并且具有stream方法，因此它提供迭代和流访问。因此，** Collection或适当的子类型通常是公共序列返回方法的最佳返回类型。** Arrays还提供了使用Arrays.asList和Stream.of方法的简单迭代和流访问。如果您返回的序列小到足以容易地放入内存中，那么最好返回一个标准集合实现，例如ArrayList或HashSet。但是**不要在内存中存储大的序列只是为了将它作为集合返回。**

If the sequence you’re returning is large but can be represented concisely, consider implementing a special-purpose collection. For example, suppose you want to return the power set of a given set, which consists of all of its subsets. The power set of {a, b, c} is {{}, {a}, {b}, {c}, {a, b}, {a, c}, {b, c}, {a, b, c}}. If a set has n elements, its power set has 2n. Therefore, you shouldn’t even consider storing the power set in a standard collection implementation. It is, however, easy to implement a custom collection for the job with the help of AbstractList.

如果您返回的序列很大但可以简洁地表示，请考虑实现一个特殊用途的集合。例如，假设您要返回给定集的幂集，该集包含其所有子集。 {a，b，c}的幂集为{{}，{a}，{b}，{c}，{a，b}，{a，c}，{b，c}，{a，b ， C}}。如果一个集合具有n个元素，则其功率集具有2n。因此，您甚至不应考虑将电源设置存储在标准集合实现中。但是，在AbstractList的帮助下，很容易为作业实现自定义集合。

The trick is to use the index of each element in the power set as a bit vector, where the nth bit in the index indicates the presence or absence of the nth element from the source set. In essence, there is a natural mapping between the binary numbers from 0 to 2n − 1 and the power set of an n-element set. Here’s the code:

技巧是使用功率集中每个元素的索引作为位向量，其中索引中的第n位指示源集合中是否存在第n个元素。本质上，从0到2n  -  1的二进制数和n元素集的幂集之间存在自然映射。这是代码：

```
// Returns the power set of an input set as custom collection
// 返回输入集的幂集作为自定义集合
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException("Set too big " + s);
        return new AbstractList<Set<E>>() {
            @Override public int size() {
                return 1 << src.size(); // 2 to the power srcSize
            }
            
            @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set)o);
            }
            
            @Override public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            } 
        };
    }
}
```

Note that PowerSet.of throws an exception if the input set has more than 30 elements. This highlights a disadvantage of using Collection as a return type rather than Stream or Iterable: Collection has an int-returning size method, which limits the length of the returned sequence to Integer.MAX_VALUE, or 231 − 1. The Collection specification does allow the size method to return 231 − 1 if the collection is larger, even infinite, but this is not a wholly satisfying solution.

请注意，如果输入集具有超过30个元素，则PowerSet.of会抛出异常。这突出了使用Collection作为返回类型而不是Stream或Iterable的缺点：Collection具有int返回大小方法，该方法将返回序列的长度限制为Integer.MAX_VALUE或231-1。Collection规范允许size方法返回231  -  1如果集合更大，甚至无限，但这不是一个完全令人满意的解决方案。

In order to write a Collection implementation atop AbstractCollection, you need implement only two methods beyond the one required for Iterable: contains and size. Often it’s easy to write efficient implementations of these methods. If it isn’t feasible, perhaps because the contents of the sequence aren’t predetermined before iteration takes place, return a stream or iterable, whichever feels more natural. If you choose, you can return both using two separate methods.

为了在AbstractCollection上编写Collection实现，您只需要实现Iterable所需的两个方法：contains和size。通常，编写这些方法的有效实现很容易。如果不可行，可能是因为在迭代发生之前未预先确定序列的内容，返回流或可迭代的，无论哪种感觉更自然。如果选择，您可以使用两种不同的方法返回。

There are times when you’ll choose the return type based solely on ease of implementation. For example, suppose you want to write a method that returns all of the (contiguous) sublists of an input list. It takes only three lines of code to generate these sublists and put them in a standard collection, but the memory required to hold this collection is quadratic in the size of the source list. While this is not as bad as the power set, which is exponential, it is clearly unacceptable. Implementing a custom collection, as we did for the power set, would be tedious, more so because the JDK lacks a skeletal Iterator implementation to help us.

有时您会根据易于实施的方式选择退货类型。例如，假设您要编写一个返回输入列表的所有（连续）子列表的方法。生成这些子列表只需要三行代码并将它们放在标准集合中，但保存此集合所需的内存是源列表大小的二次方。虽然这并不像指数的功率集那么糟糕，但显然是不可接受的。正如我们为电源设置所做的那样，实现自定义集合将是乏味的，因为JDK缺乏骨架迭代器实现来帮助我们。

It is, however, straightforward to implement a stream of all the sublists of an input list, though it does require a minor insight. Let’s call a sublist that contains the first element of a list a prefix of the list. For example, the prefixes of (a, b, c) are (a), (a, b), and (a, b, c). Similarly, let’s call a sublist that contains the last element a suffix, so the suffixes of (a, b, c) are (a, b, c), (b, c), and (c). The insight is that the sublists of a list are simply the suffixes of the prefixes (or identically, the prefixes of the suffixes) and the empty list. This observation leads directly to a clear, reasonably concise implementation:

但是，直接实现输入列表的所有子列表的流，尽管它确实需要一个小的洞察力。让我们调用一个子列表，该子列表包含列表的第一个元素和列表的前缀。例如，（a，b，c）的前缀是（a），（a，b）和（a，b，c）。类似地，让我们调用包含后缀的最后一个元素的子列表，因此（a，b，c）的后缀是（a，b，c），（b，c）和（c）。洞察力是列表的子列表只是前缀的后缀（或相同的后缀的前缀）和空列表。这一观察直接导致了一个清晰，合理简洁的实施：

```
// Returns a stream of all the sublists of its input list
// 返回其输入列表的所有子列表的流
public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list) {
        return Stream.concat(Stream.of(Collections.emptyList()),prefixes(list).flatMap(SubLists::suffixes));
    }
    
    private static <E> Stream<List<E>> prefixes(List<E> list) {
        return IntStream.rangeClosed(1, list.size()).mapToObj(end -> list.subList(0, end));
    }
    
    private static <E> Stream<List<E>> suffixes(List<E> list) {
        return IntStream.range(0, list.size()).mapToObj(start -> list.subList(start, list.size()));
    }
}
```

Note that the Stream.concat method is used to add the empty list into the returned stream. Also note that the flatMap method (Item 45) is used to generate a single stream consisting of all the suffixes of all the prefixes. Finally, note that we generate the prefixes and suffixes by mapping a stream of consecutive int values returned by IntStream.range and IntStream.rangeClosed. This idiom is, roughly speaking, the stream equivalent of the standard for-loop on integer indices. Thus, our sublist implementation is similar in spirit to the obvious nested for-loop:

请注意，Stream.concat方法用于将空列表添加到返回的流中。另请注意，flatMap方法（第45项）用于生成由所有前缀的所有后缀组成的单个流。最后，请注意我们通过映射IntStream.range和IntStream.rangeClosed返回的连续int值流来生成前缀和后缀。粗略地说，这个成语是整数索引上标准for循环的流等价物。因此，我们的子列表实现在精神上类似于明显的嵌套for循环：

```
for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end <= src.size(); end++)
        System.out.println(src.subList(start, end));
```

It is possible to translate this for-loop directly into a stream. The result is more concise than our previous implementation, but perhaps a bit less readable. It is similar in spirit to the streams code for the Cartesian product in Item 45:

可以将此for循环直接转换为流。结果比我们之前的实现更简洁，但可能性稍差。它在精神上类似于第45项中笛卡尔积的流代码：

```
// Returns a stream of all the sublists of its input list
// 返回其输入列表的所有子列表的流
public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
    .mapToObj(start ->
    IntStream.rangeClosed(start + 1, list.size())
    .mapToObj(end -> list.subList(start, end)))
    .flatMap(x -> x);
}
```

Like the for-loop that precedes it, this code does not emit the empty list. In order to fix this deficiency, you could either use concat, as we did in the previous version, or replace 1 by (int) Math.signum(start) in the rangeClosed call.

与之前的for循环一样，此代码不会发出空列表。为了解决这个问题，您可以使用concat，就像我们在之前版本中所做的那样，或者在rangeClosed调用中用（int）Math.signum（start）替换1。

Either of these stream implementations of sublists is fine, but both will require some users to employ a Stream-to-Iterable adapter or to use a stream in places where iteration would be more natural. Not only does the Stream-to- Iterable adapter clutter up client code, but it slows down the loop by a factor of 2.3 on my machine. A purpose-built Collection implementation (not shown here) is considerably more verbose but runs about 1.4 times as fast as our stream-based implementation on my machine.

这些子列表的流实现中的任何一个都很好，但两者都需要一些用户使用Stream-to-Iterable适配器或在迭代更自然的地方使用流。 Stream-to-Iterable适配器不仅使客户端代码混乱，而且还会使我的机器上的循环速度降低2.3倍。专用的Collection实现（此处未显示）相当冗长，但运行速度是我机器上基于流的实现的1.4倍。

In summary, when writing a method that returns a sequence of elements, remember that some of your users may want to process them as a stream while others may want to iterate over them. Try to accommodate both groups. If it’s feasible to return a collection, do so. If you already have the elements in a collection or the number of elements in the sequence is small enough to justify creating a new one, return a standard collection such as ArrayList. Otherwise, consider implementing a custom collection as we did for the power set. If it isn’t feasible to return a collection, return a stream or iterable, whichever seems more natural. If, in a future Java release, the Stream interface declaration is modified to extend Iterable, then you should feel free to return streams because they will allow for both stream processing and iteration.

总之，在编写返回元素序列的方法时，请记住，您的某些用户可能希望将它们作为流处理，而其他用户可能希望迭代它们。尽量适应两个群体。如果返回集合是可行的，请执行此操作。如果您已经拥有集合中的元素，或者序列中的元素数量足够小以证明创建新元素，则返回标准集合，例如ArrayList。否则，请考虑实施自定义集合，就像我们为电源设置所做的那样。如果返回集合是不可行的，则返回一个流或可迭代的，无论哪个看起来更自然。如果在将来的Java版本中，Stream接口声明被修改为扩展Iterable，那么您应该随意返回流，因为它们将允许流处理和迭代。
