## Chapter 7. Lambdas and Streams（λ表达式和流）

### Item 45: Use streams judiciously

The streams API was added in Java 8 to ease the task of performing bulk operations, sequentially or in parallel. This API provides two key abstractions: the stream, which represents a finite or infinite sequence of data elements, and the stream pipeline, which represents a multistage computation on these elements. The elements in a stream can come from anywhere. Common sources include collections, arrays, files, regular expression pattern matchers, pseudorandom number generators, and other streams. The data elements in a stream can be object references or primitive values. Three primitive types are supported: int, long, and double.

在Java 8中添加了流API，以简化顺序或并行执行批量操作的任务。该API提供了两个关键的抽象：流，表示有限或无限的数据元素序列，以及流管道，表示对这些元素的多级计算。流中的元素可以来自任何地方。常见的源包括集合，数组，文件，正则表达式模式匹配器，伪随机数生成器和其他流。流中的数据元素可以是对象引用或原始值。支持三种基本类型：int，long和double。

A stream pipeline consists of a source stream followed by zero or more intermediate operations and one terminal operation. Each intermediate operation transforms the stream in some way, such as mapping each element to a function of that element or filtering out all elements that do not satisfy some condition. Intermediate operations all transform one stream into another, whose element type may be the same as the input stream or different from it. The terminal operation performs a final computation on the stream resulting from the last intermediate operation, such as storing its elements into a collection, returning a certain element, or printing all of its elements.

流管道由源流和零个或多个中间操作以及一个终端操作组成。每个中间操作以某种方式转换流，例如将每个元素映射到该元素的函数或过滤掉不满足某些条件的所有元素。中间操作都将一个流转换为另一个流，其元素类型可以与输入流相同或与之不同。终端操作对从最后的中间操作产生的流执行最终计算，例如将其元素存储到集合中，返回某个元素或打印其所有元素。

Stream pipelines are evaluated lazily: evaluation doesn’t start until the terminal operation is invoked, and data elements that aren’t required in order to complete the terminal operation are never computed. This lazy evaluation is what makes it possible to work with infinite streams. Note that a stream pipeline without a terminal operation is a silent no-op, so don’t forget to include one.

流管道被懒惰地评估：在调用终端操作之前不开始评估，并且从不计算为完成终端操作而不需要的数据元素。这种懒惰的评估使得可以使用无限流。请注意，没有终端操作的流管道是静默无操作，因此不要忘记包含一个。

The streams API is fluent: it is designed to allow all of the calls that comprise a pipeline to be chained into a single expression. In fact, multiple pipelines can be chained together into a single expression.

流API非常流畅：它旨在允许将构成管道的所有调用链接到单个表达式中。实际上，多个管道可以链接在一起形成一个表达式。

By default, stream pipelines run sequentially. Making a pipeline execute in parallel is as simple as invoking the parallel method on any stream in the pipeline, but it is seldom appropriate to do so (Item 48).

默认情况下，流管道按顺序运行。使管道并行执行就像在管道中的任何流上调用并行方法一样简单，但很少这样做（第48项）。

The streams API is sufficiently versatile that practically any computation can be performed using streams, but just because you can doesn’t mean you should. When used appropriately, streams can make programs shorter and clearer; when used inappropriately, they can make programs difficult to read and maintain. There are no hard and fast rules for when to use streams, but there are heuristics.

流API具有足够的通用性，几乎任何计算都可以使用流来执行，但仅仅因为你并不意味着你应该这样做。如果使用得当，流可以使程序更短更清晰;如果使用不当，可能会使程序难以阅读和维护。什么时候使用流没有硬性规定，但有启发式方法。

Consider the following program, which reads the words from a dictionary file and prints all the anagram groups whose size meets a user-specified minimum. Recall that two words are anagrams if they consist of the same letters in a different order. The program reads each word from a user-specified dictionary file and places the words into a map. The map key is the word with its letters alphabetized, so the key for "staple" is "aelpst", and the key for "petals" is also "aelpst": the two words are anagrams, and all anagrams share the same alphabetized form (or alphagram, as it is sometimes known). The map value is a list containing all of the words that share an alphabetized form. After the dictionary has been processed, each list is a complete anagram group. The program then iterates through the map’s values() view and prints each list whose size meets the threshold:

考虑以下程序，该程序从字典文件中读取单词并打印其大小符合用户指定的最小值的所有anagram组。回想一下，如果两个单词由不同顺序的相同字母组成，则它们是字谜。程序从用户指定的字典文件中读取每个单词并将单词放入地图中。地图键是用字母按字母顺序排列的单词，因此“staple”的键是“aelpst”，“花瓣”的键也是“aelpst”：两个单词是anagrams，所有的anagrams共享相同的字母形式（或alphagram，因为它有时是已知的）。地图值是包含共享按字母顺序排列的形式的所有单词的列表。字典处理完毕后，每个列表都是一个完整的字谜组。然后程序遍历map的values（）视图并打印每个大小符合阈值的列表：

```
// Prints all large anagram groups in a dictionary iteratively
//迭代地在字典中打印所有大的anagram组
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word),(unused) -> new TreeSet<>()).add(word);
            }
        }
        for (Set<String> group : groups.values())
        if (group.size() >= minGroupSize)
            System.out.println(group.size() + ": " + group);
    }
    
    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

One step in this program is worthy of note. The insertion of each word into the map, which is shown in bold, uses the computeIfAbsent method, which was added in Java 8. This method looks up a key in the map: If the key is present, the method simply returns the value associated with it. If not, the method computes a value by applying the given function object to the key, associates this value with the key, and returns the computed value. The computeIfAbsent method simplifies the implementation of maps that associate multiple values with each key. 

该计划的一个步骤值得注意。将每个单词插入到地图中（以粗体显示）使用在Java 8中添加的computeIfAbsent方法。此方法在地图中查找键：如果键存在，则该方法仅返回关联的值用它。如果不是，则该方法通过将给定的函数对象应用于键来计算值，将该值与键相关联，并返回计算的值。 computeIfAbsent方法简化了将多个值与每个键相关联的映射的实现。

Now consider the following program, which solves the same problem, but makes heavy use of streams. Note that the entire program, with the exception of the code that opens the dic

现在考虑以下程序，它解决了同样的问题，但大量使用了流。请注意，除了打开字典文件的代码之外，整个程序都包含在一个表达式中。在单独的表达式中打开字典的唯一原因是允许使用try-with-resources语句，这可确保关闭字典文件：

```
// Overuse of streams - don't do this!
// 过度使用流 - 不要这样做！
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
            groupingBy(word -> word.chars().sorted()
            .collect(StringBuilder::new,(sb, c) -> sb.append((char) c),
            StringBuilder::append).toString()))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .map(group -> group.size() + ": " + group)
            .forEach(System.out::println);
        }
    }
}
```

If you find this code hard to read, don’t worry; you’re not alone. It is shorter, but it is also less readable, especially to programmers who are not experts in the use of streams. Overusing streams makes programs hard to read and maintain. Luckily, there is a happy medium. The following program solves the same problem, using streams without overusing them. The result is a program that’s both shorter and clearer than the original:

如果您发现此代码难以阅读，请不要担心;你不是一个人。它更短，但也不太可读，特别是对于不是使用流的专家的程序员。过度使用流程会使程序难以阅读和维护。幸运的是，有一个幸福的媒介。以下程序使用流而不过度使用流来解决相同的问题。结果是一个比原始程序更短更清晰的程序：

```
// Tasteful use of streams enhances clarity and conciseness
// 高雅的溪流使用增强了清晰度和简洁性
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }
    // alphabetize method is the same as in original version
}
```

Even if you have little previous exposure to streams, this program is not hard to understand. It opens the dictionary file in a try-with-resources block, obtaining a stream consisting of all the lines in the file. The stream variable is named words to suggest that each element in the stream is a word. The pipeline on this stream has no intermediate operations; its terminal operation collects all the words into a map that groups the words by their alphabetized form (Item 46). This is exactly the same map that was constructed in both previous versions of the program. Then a new Stream<List<String>> is opened on the values() view of the map. The elements in this stream are, of course, the anagram groups. The stream is filtered so that all of the groups whose size is less than minGroupSize are ignored, and finally, the remaining groups are printed by the terminal operation forEach.

即使你以前很少接触过流，这个程序也不难理解。它在try-with-resources块中打开字典文件，获取包含文件中所有行的流。 stream变量被命名为单词，表示流中的每个元素都是一个单词。此流上的管道没有中间操作;它的终端操作将所有单词收集到一个地图中，该地图按字母顺序排列单词（第46项）。这与在以前版本的程序中构建的地图完全相同。然后在地图的values（）视图上打开一个新的Stream <List <String >>。当然，这个流中的元素是anagram组。过滤流以便忽略大小小于minGroupSize的所有组，最后，通过终端操作forEach打印剩余的组。

Note that the lambda parameter names were chosen carefully. The parameter g should really be named group, but the resulting line of code would be too wide for the book. **In the absence of explicit types, careful naming of lambda parameters is essential to the readability of stream pipelines.**

请注意，小心选择了lambda参数名称。参数g应该真正命名为group，但是生成的代码行对于本书来说太宽了。 **在没有显式类型的情况下，仔细命名lambda参数对于流管道的可读性至关重要。**

Note also that word alphabetization is done in a separate alphabetize method. This enhances readability by providing a name for the operation and keeping implementation details out of the main program. **Using helper methods is even more important for readability in stream pipelines than in iterative code** because pipelines lack explicit type information and named temporary variables.

另请注意，单词字母化是在单独的字母顺序排列方法中完成的。这通过提供操作的名称并将实现细节保留在主程序之外来增强可读性。 **使用辅助方法对于流管道中的可读性比迭代代码**更重要，因为管道缺少显式类型信息和命名临时变量。

The alphabetize method could have been reimplemented to use streams, but a stream-based alphabetize method would have been less clear, more difficult to write correctly, and probably slower. These deficiencies result from Java’s lack of support for primitive char streams (which is not to imply that Java should have supported char streams; it would have been infeasible to do so). To demonstrate the hazards of processing char values with streams, consider the following code:

可以重新实现字母顺序排列方法以使用流，但是基于流的字母顺序排列方法不太清晰，更难以正确编写，并且可能更慢。这些缺陷是由于Java缺乏对原始字符串流的支持（这并不意味着Java应该支持char流;这样做是不可行的）。要演示使用流处理char值的危险，请考虑以下代码：

```
"Hello world!".chars().forEach(System.out::print);
```

You might expect it to print Hello world!, but if you run it, you’ll find that it prints 721011081081113211911111410810033. This happens because the elements of the stream returned by "Hello world!".chars() are not char values but int values, so the int overloading of print is invoked. It is admittedly confusing that a method named chars returns a stream of int values. You could fix the program by using a cast to force the invocation of the correct overloading:

您可能希望它打印Hello world！，但如果您运行它，您会发现它打印721011081081113211911111410810033。这是因为“Hello world！”。chars（）返回的流的元素不是char值而是int值，因此调用print的int重载。令人遗憾的是，名为chars的方法返回一个int值流。您可以通过使用强制转换来强制调用正确的重载来修复程序：

```
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```

but ideally you should refrain from using streams to process char values. When you start using streams, you may feel the urge to convert all your loops into streams, but resist the urge. While it may be possible, it will likely harm the readability and maintainability of your code base. As a rule, even moderately complex tasks are best accomplished using some combination of streams and iteration, as illustrated by the Anagrams programs above. So **refactor existing code to use streams and use them in new code only where it makes sense to do so.**

但理想情况下，您应该避免使用流来处理char值。当您开始使用流时，您可能会感觉到将所有循环转换为流的冲动，但抵制冲动。尽管有可能，但可能会损害代码库的可读性和可维护性。通常，使用流和迭代的某种组合可以最好地完成中等复杂的任务，如上面的Anagrams程序所示。因此**重构现有代码以使用流，并仅在有意义的情况下在新代码中使用它们。**

As shown in the programs in this item, stream pipelines express repeated computation using function objects (typically lambdas or method references), while iterative code expresses repeated computation using code blocks. There are some things you can do from code blocks that you can’t do from function objects:

如该项目中的程序所示，流管道使用函数对象（通常是lambdas或方法引用）表示重复计算，而迭代代码使用代码块表示重复计算。您可以从函数对象无法执行的代码块中执行以下操作：

- From a code block, you can read or modify any local variable in scope; from a lambda, you can only read final or effectively final variables [JLS 4.12.4], and you can’t modify any local variables.

- 从代码块中，您可以读取或修改范围内的任何局部变量;从lambda中，您只能读取最终或有效的最终变量[JLS 4.12.4]，并且您无法修改任何局部变量。

- From a code block, you can return from the enclosing method, break or continue an enclosing loop, or throw any checked exception that this method is declared to throw; from a lambda you can do none of these things.

- 从代码块，您可以从封闭方法返回，中断或继续封闭循环，或抛出声明此方法被抛出的任何已检查异常;从一个lambda你不能做这些事情。

If a computation is best expressed using these techniques, then it’s probably not a good match for streams. Conversely, streams make it very easy to do some things:

如果使用这些技术最好地表达计算，那么它可能不是流的良好匹配。相反，流可以很容易地做一些事情：

- Uniformly transform sequences of elements

 - 均匀地转换元素序列

- Filter sequences of elements

 - 过滤元素序列

- Combine sequences of elements using a single operation (for example to add them, concatenate them, or compute their minimum)

- 使用单个操作组合元素序列（例如，添加它们，连接它们或计算它们的最小值）

- Accumulate sequences of elements into a collection, perhaps grouping them by some common attribute

 - 将元素序列累积到集合中，或者通过一些常见属性对它们进行分组

- Search a sequence of elements for an element satisfying some criterion

 - 在元素序列中搜索满足某个标准的元素

If a computation is best expressed using these techniques, then it is a good candidate for streams.

如果使用这些技术最好地表达计算，那么它是流的良好候选者。

One thing that is hard to do with streams is to access corresponding elements from multiple stages of a pipeline simultaneously: once you map a value to some other value, the original value is lost. One workaround is to map each value to a pair object containing the original value and the new value, but this is not a satisfying solution, especially if the pair objects are required for multiple stages of a pipeline. The resulting code is messy and verbose, which defeats a primary purpose of streams. When it is applicable, a better workaround is to invert the mapping when you need access to the earlier-stage value.

使用流很难做的一件事是同时从管道的多个阶段访问相应的元素：一旦将值映射到某个其他值，原始值就会丢失。一种解决方法是将每个值映射到包含原始值和新值的对对象，但这不是一个令人满意的解决方案，尤其是如果管道的多个阶段需要对对象。由此产生的代码是混乱和冗长的，这破坏了流的主要目的。如果适用，更好的解决方法是在需要访问早期阶段值时反转映射。

For example, let’s write a program to print the first twenty Mersenne primes. To refresh your memory, a Mersenne number is a number of the form 2p − 1. If p is prime, the corresponding Mersenne number may be prime; if so, it’s a Mersenne prime. As the initial stream in our pipeline, we want all the prime numbers. Here’s a method to return that (infinite) stream. We assume a static import has been used for easy access to the static members of BigInteger:

例如，让我们编写一个程序来打印前20个Mersenne素数。为了刷新你的记忆，梅森数是一个2p  -  1的数字。如果p是素数，相应的梅森数可能是素数;如果是这样的话，那就是梅森素数。作为我们管道中的初始流，我们需要所有素数。这是一种返回该（无限）流的方法。我们假设使用静态导入来轻松访问BigInteger的静态成员：

```
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

The name of the method (primes) is a plural noun describing the elements of the stream. This naming convention is highly recommended for all methods that return streams because it enhances the readability of stream pipelines. The method uses the static factory Stream.iterate, which takes two parameters: the first element in the stream, and a function to generate the next element in the stream from the previous one. Here is the program to print the first twenty Mersenne primes:

方法（primes）的名称是描述流的元素的复数名词。强烈建议所有返回流的方法使用此命名约定，因为它增强了流管道的可读性。该方法使用静态工厂Stream.iterate，它接受两个参数：流中的第一个元素，以及从前一个元素生成流中的下一个元素的函数。这是打印前20个Mersenne素数的程序：

```
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
    .filter(mersenne -> mersenne.isProbablePrime(50))
    .limit(20)
    .forEach(System.out::println);
}
```

This program is a straightforward encoding of the prose description above: it starts with the primes, computes the corresponding Mersenne numbers, filters out all but the primes (the magic number 50 controls the probabilistic primality test), limits the resulting stream to twenty elements, and prints them out.

这个程序是上面的散文描述的直接编码：它从素数开始，计算相应的梅森数，过滤掉除素数之外的所有数字（幻数50控制概率素性测试），将得到的流限制为20个元素，并打印出来。

Now suppose that we want to precede each Mersenne prime with its exponent (p). This value is present only in the initial stream, so it is inaccessible in the terminal operation, which prints the results. Luckily, it’s easy to compute the exponent of a Mersenne number by inverting the mapping that took place in the first intermediate operation. The exponent is simply the number of bits in the binary representation, so this terminal operation generates the desired result:

现在假设我们想要在每个Mersenne素数之前加上它的指数（p）。该值仅出现在初始流中，因此在终端操作中无法访问，从而打印结果。幸运的是，通过反转第一个中间操作中发生的映射，可以很容易地计算出Mersenne数的指数。指数只是二进制表示中的位数，因此该终端操作会生成所需的结果：

```
.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));
```

There are plenty of tasks where it is not obvious whether to use streams or iteration. For example, consider the task of initializing a new deck of cards. Assume that Card is an immutable value class that encapsulates a Rank and a Suit, both of which are enum types. This task is representative of any task that requires computing all the pairs of elements that can be chosen from two sets. Mathematicians call this the Cartesian product of the two sets. Here’s an iterative implementation with a nested for-each loop that should look very familiar to you:

有很多任务，无论是使用流还是迭代都不明显。例如，考虑初始化一副新牌的任务。假设Card是一个不可变的值类，它封装了Rank和Suit，两者都是枚举类型。此任务代表任何需要计算可从两组中选择的所有元素对的任务。数学家称之为两组的笛卡尔积。这是一个带有嵌套for-each循环的迭代实现，对你来说应该非常熟悉：

```
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
    for (Rank rank : Rank.values())
    result.add(new Card(suit, rank));
    return result;
}
```

And here is a stream-based implementation that makes use of the intermediate operation flatMap. This operation maps each element in a stream to a stream and then concatenates all of these new streams into a single stream (or flattens them). Note that this implementation contains a nested lambda, shown in boldface:

这是一个基于流的实现，它使用了中间操作flatMap。此操作将流中的每个元素映射到流，然后将所有这些新流连接成单个流（或展平它们）。请注意，此实现包含嵌套的lambda，以粗体显示：

```
// Stream-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
    .flatMap(suit ->Stream.of(Rank.values())
    .map(rank -> new Card(suit, rank)))
    .collect(toList());
}
```

Which of the two versions of newDeck is better? It boils down to personal preference and the environment in which you’re programming. The first version is simpler and perhaps feels more natural. A larger fraction of Java programmers will be able to understand and maintain it, but some programmers will feel more comfortable with the second (stream-based) version. It’s a bit more concise and not too difficult to understand if you’re reasonably well-versed in streams and functional programming. If you’re not sure which version you prefer, the iterative version is probably the safer choice. If you prefer the stream version and you believe that other programmers who will work with the code will share your preference, then you should use it.

newDeck的两个版本中哪一个更好？它归结为个人偏好和您编程的环境。第一个版本更简单，也许感觉更自然。大部分Java程序员将能够理解和维护它，但是一些程序员会对第二个（基于流的）版本感觉更舒服。如果您对流和函数式编程有相当的精通，那么它会更简洁，也不会太难理解。如果您不确定自己喜欢哪个版本，则迭代版本可能是更安全的选择。如果你更喜欢流版本，并且你相信其他使用代码的程序员会分享你的偏好，那么你应该使用它。

In summary, some tasks are best accomplished with streams, and others with iteration. Many tasks are best accomplished by combining the two approaches. There are no hard and fast rules for choosing which approach to use for a task, but there are some useful heuristics. In many cases, it will be clear which approach to use; in some cases, it won’t. If you’re not sure whether a task is better served by streams or iteration, try both and see which works better.

总之，一些任务最好用流完成，其他任务最好用迭代完成。通过组合这两种方法可以最好地完成许多任务。选择哪种方法用于任务没有硬性规定，但有一些有用的启发式方法。在许多情况下，将清楚使用哪种方法;在某些情况下，它不会。如果您不确定某个任务是否更适合流或迭代，请尝试两者并查看哪个更好。
