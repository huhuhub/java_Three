## Chapter 7. Lambdas and Streams（λ表达式和流）

### Item 42: Prefer lambdas to anonymous classes

Historically, interfaces (or, rarely, abstract classes) with a single abstract method were used as function types. Their instances, known as function objects, represent functions or actions. Since JDK 1.1 was released in 1997, the primary means of creating a function object was the anonymous class (Item 24). Here’s a code snippet to sort a list of strings in order of length, using an anonymous class to create the sort’s comparison function (which imposes the sort order):

从历史上看，使用单个抽象方法的接口（或很少是抽象类）被用作函数类型。它们的实例称为函数对象，代表函数或动作。自JDK 1.1于1997年发布以来，创建函数对象的主要方法是匿名类（第24项）。这是一个代码片段，用于按长度顺序对字符串列表进行排序，使用匿名类创建排序的比较函数（强制排序顺序）：
```
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

Anonymous classes were adequate for the classic objected-oriented design patterns requiring function objects, notably the Strategy pattern [Gamma95]. The Comparator interface represents an abstract strategy for sorting; the anonymous class above is a concrete strategy for sorting strings. The verbosity of anonymous classes, however, made functional programming in Java an unappealing prospect.

In Java 8, the language formalized the notion that interfaces with a single abstract method are special and deserve special treatment. These interfaces are now known as functional interfaces, and the language allows you to create instances of these interfaces using lambda expressions, or lambdas for short. Lambdas are similar in function to anonymous classes, but far more concise. Here’s how the code snippet above looks with the anonymous class replaced by a lambda. The boilerplate is gone, and the behavior is clearly evident:

匿名类适用于需要功能对象的经典的面向对象的设计模式，特别是策略模式[Gamma95]。 Comparator接口表示用于排序的抽象策略;上面的匿名类是排序字符串的具体策略。然而，匿名类的冗长使得Java中的函数式编程成为一个没有吸引力的前景。在Java 8中，该语言形式化了这样一种概念，即使用单一抽象方法的接口是特殊的，值得特别对待。这些接口现在称为功能接口，该语言允许您使用lambda表达式或简称lambdas创建这些接口的实例。 Lambdas在功能上与匿名类相似，但更加简洁。以下是上面的代码片段如何将匿名类替换为lambda。样板消失了，行为很明显：
```
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words,(s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

Note that the types of the lambda (Comparator<String>), of its parameters (s1 and s2, both String), and of its return value (int) are not present in the code. The compiler deduces these types from context, using a process known as type inference. In some cases, the compiler won’t be able to determine the types, and you’ll have to specify them. The rules for type inference are complex: they take up an entire chapter in the JLS [JLS, 18]. Few programmers understand these rules in detail, but that’s OK. **Omit the types of all lambda parameters unless their presence makes your program clearer.** If the compiler generates an error telling you it can’t infer the type of a lambda parameter, then specify it. Sometimes you may have to cast the return value or the entire lambda expression, but this is rare.

One caveat should be added concerning type inference. Item 26 tells you not to use raw types, Item 29 tells you to favor generic types, and Item 30 tells you to favor generic methods. This advice is doubly important when you’re using lambdas, because the compiler obtains most of the type information that allows it to perform type inference from generics. If you don’t provide this information, the compiler will be unable to do type inference, and you’ll have to specify types manually in your lambdas, which will greatly increase their verbosity. By way of example, the code snippet above won’t compile if the variable words is declared to be of the raw type List instead of the parameterized type List<String>.

Incidentally, the comparator in the snippet can be made even more succinct if a comparator construction method is used in place of a lambda (Items 14. 43):

请注意，lambda（Comparator <String>）的类型，其参数（s1和s2，两个String）及其返回值（int）的类型不在代码中。编译器使用称为类型推断的过程从上下文中推导出这些类型。在某些情况下，编译器将无法确定类型，您必须指定它们。类型推断的规则很复杂：它们占据了JLS的整个章节[JLS，18]。很少有程序员详细了解这些规则，但这没关系。 **省略所有lambda参数的类型，除非它们的存在使程序更清晰。**如果编译器生成错误，告诉你它无法推断lambda参数的类型，那么指定它。有时您可能必须转换返回值或整个lambda表达式，但这种情况很少见。关于类型推断，应该添加一个警告。第26项告诉您不要使用原始类型，第29项告诉您支持泛型类型，第30项告诉您支持通用方法。当你使用lambdas时，这个建议是非常重要的，因为编译器获得了允许它从泛型执行类型推断的大多数类型信息。如果您不提供此信息，编译器将无法进行类型推断，您必须在lambdas中手动指定类型，这将大大增加它们的详细程度。举例来说，如果变量词被声明为原始类型List而不是参数化类型List <String>，则上面的代码片段将不会编译。顺便提一下，如果使用比较器构造方法代替lambda，则片段中的比较器可以更简洁（第14. 43项）：
```
Collections.sort(words, comparingInt(String::length));
```

In fact, the snippet can be made still shorter by taking advantage of the sort method that was added to the List interface in Java 8:

实际上，通过利用Java 8中添加到List接口的sort方法，可以使代码段更短：
```
words.sort(comparingInt(String::length));
```

The addition of lambdas to the language makes it practical to use function objects where it would not previously have made sense. For example, consider the Operation enum type in Item 34. Because each enum required different behavior for its apply method, we used constant-specific class bodies and overrode the apply method in each enum constant. To refresh your memory, here is the code:

将lambda添加到语言中使得使用函数对象变得切实可行。例如，考虑第34项中的Operation枚举类型。因为每个枚举对其apply方法需要不同的行为，所以我们使用常量特定的类主体并覆盖每个枚举常量中的apply方法。为了刷新你的记忆，这里是代码：
```
// Enum type with constant-specific class bodies & data (Item 34)
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    
    private final String symbol;
    
    Operation(String symbol) { this.symbol = symbol; }
    
    @Override public String toString() { return symbol; }
    
    public abstract double apply(double x, double y);
}
```

Item 34 says that enum instance fields are preferable to constant-specific class bodies. Lambdas make it easy to implement constant-specific behavior using the former instead of the latter. Merely pass a lambda implementing each enum constant’s behavior to its constructor. The constructor stores the lambda in an instance field, and the apply method forwards invocations to the lambda. The resulting code is simpler and clearer than the original version:

第34项说enum实例字段比常量特定的类体更可取。使用前者而不是后者，Lambdas可以轻松实现常量特定的行为。只需将实现每个枚举常量行为的lambda传递给它的构造函数。构造函数将lambda存储在实例字段中，apply方法将调用转发给lambda。生成的代码比原始版本更简单，更清晰：
```
// Enum with function object fields & constant-specific behavior
public enum Operation {
    PLUS ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);
    
    private final String symbol;
    
    private final DoubleBinaryOperator op;
    
    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }
    
    @Override public String toString() { return symbol; }
    
    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

Note that we’re using the DoubleBinaryOperator interface for the lambdas that represent the enum constant’s behavior. This is one of the many predefined functional interfaces in java.util.function (Item 44). It represents a function that takes two double arguments and returns a double result.

Looking at the lambda-based Operation enum, you might think constantspecific method bodies have outlived their usefulness, but this is not the case. Unlike methods and classes, **lambdas lack names and documentation; if a computation isn’t self-explanatory, or exceeds a few lines, don’t put it in a lambda.** One line is ideal for a lambda, and three lines is a reasonable maximum. If you violate this rule, it can cause serious harm to the readability of your programs. If a lambda is long or difficult to read, either find a way to simplify it or refactor your program to eliminate it. Also, the arguments passed to enum constructors are evaluated in a static context. Thus, lambdas in enum constructors can’t access instance members of the enum. Constant-specific class bodies are still the way to go if an enum type has constant-specific behavior that is difficult to understand, that can’t be implemented in a few lines, or that requires access to instance fields or methods.

Likewise, you might think that anonymous classes are obsolete in the era of lambdas. This is closer to the truth, but there are a few things you can do with anonymous classes that you can’t do with lambdas. Lambdas are limited to functional interfaces. If you want to create an instance of an abstract class, you can do it with an anonymous class, but not a lambda. Similarly, you can use anonymous classes to create instances of interfaces with multiple abstract methods. Finally, a lambda cannot obtain a reference to itself. In a lambda, the this keyword refers to the enclosing instance, which is typically what you want. In an anonymous class, the this keyword refers to the anonymous class instance. If you need access to the function object from within its body, then you must use an anonymous class.

Lambdas share with anonymous classes the property that you can’t reliably serialize and deserialize them across implementations. Therefore, **you should rarely, if ever, serialize a lambda** (or an anonymous class instance). If you have a function object that you want to make serializable, such as a Comparator, use an instance of a private static nested class (Item 24).

In summary, as of Java 8, lambdas are by far the best way to represent small function objects. **Don’t use anonymous classes for function objects unless you have to create instances of types that aren’t functional interfaces. Also, remember that lambdas make it so easy to represent small function objects that it opens the door to functional programming techniques that were not previously practical in Java.

请注意，我们使用DoubleBinaryOperator接口来表示枚举常量行为的lambdas。这是java.util.function（Item 44）中许多预定义的功能接口之一。它表示一个函数，它接受两个双参数并返回一个double结果。查看基于lambda的Operation枚举，您可能会认为常量特定的方法体已经过时了，但事实并非如此。与方法和类不同，** lambdas缺少名称和文档;如果计算不是不言自明的，或超过几行，则不要将它放在lambda中。**一行是lambda的理想选择，三行是合理的最大值。如果违反此规则，可能会严重损害程序的可读性。如果lambda很长或难以阅读，要么找到简化它的方法，要么重构你的程序以消除它。此外，传递给枚举构造函数的参数在静态上下文中进行计算。因此，枚举构造函数中的lambdas无法访问枚举的实例成员。如果枚举类型具有难以理解的常量特定行为，无法在几行中实现，或者需要访问实例字段或方法，则仍然可以使用特定于常量的类主体。同样，您可能会认为匿名类在lambdas时代已经过时了。这更接近事实，但是你可以用匿名类做一些事情，而你无法用lambdas做。 Lambdas仅限于功能接口。如果要创建抽象类的实例，可以使用匿名类，但不能使用lambda。同样，您可以使用匿名类来创建具有多个抽象方法的接口实例。最后，lambda无法获得对自身的引用。在lambda中，this关键字引用封闭实例，这通常是您想要的。在匿名类中，this关键字引用匿名类实例。如果需要从其体内访问函数对象，则必须使用匿名类。 Lambdas与匿名类共享您无法在实现中可靠地序列化和反序列化它们的属性。因此，**您应该很少（如果有的话）序列化lambda **（或匿名类实例）。如果您有一个要进行序列化的函数对象，例如Comparator，请使用私有静态嵌套类的实例（Item 24）。总之，从Java 8开始，lambda是迄今为止表示小函数对象的最佳方式。 **除非必须创建非功能接口类型的实例，否则不要对函数对象使用匿名类。另外，请记住lambda使代表小函数对象变得如此容易，以至于它打开了以前在Java中不实用的函数式编程技术的大门。

