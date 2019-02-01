## Chapter 7. Lambdas and Streams（λ表达式和流）

### Item 44: Favor the use of standard functional interfaces

Now that Java has lambdas, best practices for writing APIs have changed considerably. For example, the Template Method pattern [Gamma95], wherein a subclass overrides a primitive method to specialize the behavior of its superclass, is far less attractive. The modern alternative is to provide a static factory or constructor that accepts a function object to achieve the same effect. More generally, you’ll be writing more constructors and methods that take function objects as parameters. Choosing the right functional parameter type demands care.

既然Java有lambda，那么编写API的最佳实践已经发生了很大变化。例如，模板方法模式[Gamma95]，其中子类重写基本方法以专门化其超类的行为，远没那么有吸引力。现代的替代方法是提供一个静态工厂或构造函数，它接受一个函数对象来实现相同的效果。更一般地说，您将编写更多以函数对象作为参数的构造函数和方法。选择正确的功能参数类型需要谨慎。

Consider LinkedHashMap. You can use this class as a cache by overriding its protected removeEldestEntry method, which is invoked by put each time a new key is added to the map. When this method returns true, the map removes its eldest entry, which is passed to the method. The following override allows the map to grow to one hundred entries and then deletes the eldest entry each time a new key is added, maintaining the hundred most recent entries:

考虑LinkedHashMap。您可以通过覆盖其受保护的removeEldestEntry方法将此类用作缓存，该方法每次将新键添加到地图时都会调用。当此方法返回true时，映射将删除其最旧的条目，该条目将传递给该方法。以下覆盖允许地图增长到一百个条目，然后在每次添加新密钥时删除最旧的条目，保留最近的一百个条目：

```
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > 100;
}
```

This technique works fine, but you can do much better with lambdas. If LinkedHashMap were written today, it would have a static factory or constructor that took a function object. Looking at the declaration for removeEldestEntry, you might think that the function object should take a Map.Entry<K,V> and return a boolean, but that wouldn’t quite do it: The removeEldestEntry method calls size() to get the number of entries in the map, which works because removeEldestEntry is an instance method on the map. The function object that you pass to the constructor is not an instance method on the map and can’t capture it because the map doesn’t exist yet when its factory or constructor is invoked. Thus, the map must pass itself to the function object, which must therefore take the map on input as well as its eldest entry. If you were to declare such a functional interface, it would look something like this:

这种技术很好，但你可以用lambdas做得更好。如果今天写了LinkedHashMap，它将有一个带有函数对象的静态工厂或构造函数。查看removeEldestEntry的声明，你可能会认为函数对象应该采用Map.Entry <K，V>并返回一个布尔值，但是不会这样做：removeEldestEntry方法调用size（）来获取数字地图中的条目，因为removeEldestEntry是地图上的实例方法。传递给构造函数的函数对象不是地图上的实例方法，并且无法捕获它，因为在调用其工厂或构造函数时映射尚不存在。因此，映射必须将自身传递给函数对象，因此函数对象必须在输入及其最旧条目上获取映射。如果你要声明这样一个功能界面，它看起来像这样：

```
// Unnecessary functional interface; use a standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```

This interface would work fine, but you shouldn’t use it, because you don’t need to declare a new interface for this purpose. The java.util.function package provides a large collection of standard functional interfaces for your use. **If one of the standard functional interfaces does the job, you should generally use it in preference to a purpose-built functional interface.** This will make your API easier to learn, by reducing its conceptual surface area, and will provide significant interoperability benefits, as many of the standard functional interfaces provide useful default methods. The Predicate interface, for instance, provides methods to combine predicates. In the case of our LinkedHashMap example, the standard BiPredicate<Map<K,V>, Map.Entry<K,V>> interface should be used in preference to a custom EldestEntryRemovalFunction interface.

此接口可以正常工作，但您不应该使用它，因为您不需要为此目的声明新接口。 java.util.function包提供了大量标准功能接口供您使用。 **如果其中一个标准功能接口完成了这项工作，您通常应该优先使用它而不是专门构建的功能接口。**这将使您的API更容易学习，通过减少其概念表面积，并将提供重要的互操作性的好处，因为许多标准功能接口提供了有用的默认方法。例如，Predicate接口提供了组合谓词的方法。对于LinkedHashMap示例，应优先使用标准BiPredicate <Map <K，V>，Map.Entry <K，V >>接口，而不是自定义EldestEntryRemovalFunction接口。

There are forty-three interfaces in java.util.Function. You can’t be expected to remember them all, but if you remember six basic interfaces, you can derive the rest when you need them. The basic interfaces operate on object reference types. The Operator interfaces represent functions whose result and argument types are the same. The Predicate interface represents a function that takes an argument and returns a boolean. The Function interface represents a function whose argument and return types differ. The Supplier interface represents a function that takes no arguments and returns (or “supplies”) a value. Finally, Consumer represents a function that takes an argument and returns nothing, essentially consuming its argument. The six basic functional interfaces are summarized below:

java.util.Function中有四十三个接口。不能指望你记住它们，但如果你记得六个基本接口，你可以在需要时得到其余的接口。基本接口对对象引用类型进行操作。 Operator接口表示结果和参数类型相同的函数。 Predicate接口表示一个接受参数并返回布尔值的函数。 Function接口表示其参数和返回类型不同的函数。 Supplier接口表示不带参数并返回（或“提供”）值的函数。最后，Consumer表示一个函数，它接受一个参数并且什么都不返回，基本上消耗它的参数。六个基本功能接口总结如下：

|    Interface    |       Function Signature       |      Example     |
|:-------:|:-------:|:-------:|
|   UnaryOperator<T>  |     T apply(T t)    |   String::toLowerCase   |
|   BinaryOperator<T>  |     T apply(T t1, T t2)    |   BigInteger::add   |
|   Predicate<T>  |     boolean test(T t)    |   Collection::isEmpty   |
|   Function<T,R>  |     R apply(T t)    |   Arrays::asList   |
|   Supplier<T>  |     T get()    |   Instant::now   |
|   Consumer<T>  |     void accept(T t)    |   System.out::println   |

There are also three variants of each of the six basic interfaces to operate on the primitive types int, long, and double. Their names are derived from the basic interfaces by prefixing them with a primitive type. So, for example, a predicate that takes an int is an IntPredicate, and a binary operator that takes two long values and returns a long is a LongBinaryOperator. None of these variant types is parameterized except for the Function variants, which are parameterized by return type. For example, LongFunction<int[]> takes a long and returns an int[].

六种基本接口中的每一种都有三种变体可以对基本类型int，long和double进行操作。它们的名称来源于基本接口，前缀为基本类型。因此，例如，带有int的谓词是IntPredicate，带有两个long值并返回long的二元运算符是LongBinaryOperator。除函数变量外，这些变量类型都不参数化，函数变量由返回类型参数化。例如，LongFunction <int []>接受一个long并返回一个int []。

There are nine additional variants of the Function interface, for use when the result type is primitive. The source and result types always differ, because a function from a type to itself is a UnaryOperator. If both the source and result types are primitive, prefix Function with SrcToResult, for example LongToIntFunction (six variants). If the source is a primitive and the result is an object reference, prefix Function with <Src>ToObj, for example DoubleToObjFunction (three variants).

Function接口有九个附加变体，供结果类型为原始时使用。源和结果类型总是不同，因为从类型到自身的函数是UnaryOperator。如果源类型和结果类型都是原始类型，则使用SrcToResult作为前缀Function，例如LongToIntFunction（六个变体）。如果源是基元并且结果是对象引用，则使用<Src> ToObj作为前缀Function，例如DoubleToObjFunction（三个变体）。

There are two-argument versions of the three basic functional interfaces for which it makes sense to have them: BiPredicate<T,U>, BiFunction<T,U,R>, and BiConsumer<T,U>. There are also BiFunction variants returning the three relevant primitive types: ToIntBiFunction<T,U>, ToLongBiFunction<T,U>, and ToDoubleBiFunction<T,U>. There are two-argument variants of Consumer that take one object reference and one primitive type: ObjDoubleConsumer<T>, ObjIntConsumer<T>, and ObjLongConsumer<T>. In total, there are nine two-argument versions of the basic interfaces.

有三个基本功能接口的两个参数版本，使用它们是有意义的：BiPredicate <T，U>，BiFunction <T，U，R>和BiConsumer <T，U>。还有BiFunction变体返回三种相关的基本类型：ToIntBiFunction <T，U>，ToLongBiFunction <T，U>和ToDoubleBiFunction <T，U>。 Consumer的两个参数变体采用一个对象引用和一个基本类型：ObjDoubleConsumer <T>，ObjIntConsumer <T>和ObjLongConsumer <T>。总共有九个基本接口的双参数版本。

Finally, there is the BooleanSupplier interface, a variant of Supplier that returns boolean values. This is the only explicit mention of the boolean type in any of the standard functional interface names, but boolean return values are supported via Predicate and its four variant forms. The BooleanSupplier interface and the forty-two interfaces described in the previous paragraphs account for all forty-three standard functional interfaces. Admittedly, this is a lot to swallow, and not terribly orthogonal. On the other hand, the bulk of the functional interfaces that you’ll need have been written for you and their names are regular enough that you shouldn’t have too much trouble coming up with one when you need it.

最后，还有BooleanSupplier接口，这是Supplier的一个变量，它返回布尔值。这是任何标准功能接口名称中唯一明确提到的布尔类型，但是通过Predicate及其四种变体形式支持布尔返回值。 BooleanSupplier接口和前面段落中描述的四十二个接口占所有四十三个标准功能接口。不可否认，这是一个很大的吞并，而不是非常正交。另一方面，您需要的大部分功能接口都是为您编写的，并且它们的名称足够常规，以便您在需要时不会遇到太多麻烦。

Most of the standard functional interfaces exist only to provide support for primitive types. **Don’t be tempted to use basic functional interfaces with boxed primitives instead of primitive functional interfaces.** While it works, it violates the advice of Item 61, “prefer primitive types to boxed primitives.” The performance consequences of using boxed primitives for bulk operations can be deadly.

大多数标准功能接口仅用于提供对原始类型的支持。 **不要试图使用基本功能接口与盒装基元而不是原始功能接口。**虽然它有效，但它违反了第61条的建议，“更喜欢原始类型到盒装基元。”使用盒装的性能后果批量操作的原语可能是致命的。

Now you know that you should typically use standard functional interfaces in preference to writing your own. But when should you write your own? Of course you need to write your own if none of the standard ones does what you need, for example if you require a predicate that takes three parameters, or one that throws a checked exception. But there are times you should write your own functional interface even when one of the standard ones is structurally identical.

现在您知道通常应该使用标准功能接口而不是编写自己的接口。但你应该什么时候写自己的？当然，如果没有标准的那些符合您的需要，您需要自己编写，例如，如果您需要一个带有三个参数的谓词，或者一个抛出已检查异常的谓词。但有时您应该编写自己的功能界面，即使其中一个标准结构完全相同。

Consider our old friend Comparator<T>, which is structurally identical to the ToIntBiFunction<T,T> interface. Even if the latter interface had existed when the former was added to the libraries, it would have been wrong to use it. There are several reasons that Comparator deserves its own interface. First, its name provides excellent documentation every time it is used in an API, and it’s used a lot. Second, the Comparator interface has strong requirements on what constitutes a valid instance, which comprise its general contract. By implementing the interface, you are pledging to adhere to its contract. Third, the interface is heavily outfitted with useful default methods to transform and combine comparators.

考虑我们的老朋友Comparator <T>，它在结构上与ToIntBiFunction <T，T>接口相同。即使后者接口已经存在，当前者被添加到库中时，使用它也是错误的。 Comparator有几个原因值得拥有自己的界面。首先，它的名称在每次在API中使用时都提供了出色的文档，并且它被大量使用。其次，Comparator接口对构成有效实例的内容有很强的要求，有效实例包含其一般合同。通过实施界面，您承诺遵守其合同。第三，接口配备了大量有用的默认方法来转换和组合比较器。

You should seriously consider writing a purpose-built functional interface in preference to using a standard one if you need a functional interface that shares one or more of the following characteristics with Comparator:

如果您需要一个与Comparator共享以下一个或多个特性的功能接口，您应该认真考虑编写专用的功能接口而不是使用标准接口：

- It will be commonly used and could benefit from a descriptive name.

它将被普遍使用，并可从描述性名称中受益。

- It has a strong contract associated with it.

它与它有很强的合同。

- It would benefit from custom default methods.

 - 它将受益于自定义默认方法

If you elect to write your own functional interface, remember that it’s an interface and hence should be designed with great care (Item 21).

如果您选择编写自己的功能界面，请记住它是一个界面，因此应该非常谨慎地设计（第21项）。

Notice that the EldestEntryRemovalFunction interface (page 199) is labeled with the @FunctionalInterface annotation. This annotation type is similar in spirit to @Override. It is a statement of programmer intent that serves three purposes: it tells readers of the class and its documentation that the interface was designed to enable lambdas; it keeps you honest because the interface won’t compile unless it has exactly one abstract method; and it prevents maintainers from accidentally adding abstract methods to the interface as it evolves. **Always annotate your functional interfaces with the @FunctionalInterface annotation.**

请注意，EldestEntryRemovalFunction接口（第199页）标有@FunctionalInterface注释。此注释类型在精神上与@Override类似。它是程序员意图的声明，有三个目的：它告诉读者该类及其文档，该接口旨在启用lambdas;它保持诚实，因为除非它只有一个抽象方法，否则接口不会编译;并且它可以防止维护者在接口发生时意外地将抽象方法添加到接口。 **始终使用@FunctionalInterface注释注释您的功能接口。**

A final point should be made concerning the use of functional interfaces in APIs. Do not provide a method with multiple overloadings that take different functional interfaces in the same argument position if it could create a possible ambiguity in the client. This is not just a theoretical problem. The submit method of ExecutorService can take either a Callable<T> or a Runnable, and it is possible to write a client program that requires a cast to indicate the correct overloading (Item 52). The easiest way to avoid this problem is not to write overloadings that take different functional interfaces in the same argument position. This is a special case of the advice in Item 52, “use overloading judiciously.”

关于API中功能接口的使用，应该最后一点。如果可能在客户端中产生可能的歧义，则不要提供具有多个重载的方法，这些方法在相同的参数位置采用不同的功能接口。这不仅仅是一个理论问题。 ExecutorService的submit方法可以采用Callable <T>或Runnable，并且可以编写一个需要强制转换的客户端程序来指示正确的重载（第52项）。避免此问题的最简单方法是不要编写在同一参数位置使用不同功能接口的重载。这是第52项建议中的一个特例，“明智地使用重载”。

In summary, now that Java has lambdas, it is imperative that you design your APIs with lambdas in mind. Accept functional interface types on input and return them on output. It is generally best to use the standard interfaces provided in java.util.function.Function, but keep your eyes open for the relatively rare cases where you would be better off writing your own functional interface.

总而言之，既然Java已经有了lambdas，那么在设计API时必须考虑到lambdas。接受输入上的功能接口类型并在输出上返回它们。通常最好使用java.util.function.Function中提供的标准接口，但请注意相对罕见的情况，即最好编写自己的功能接口。
