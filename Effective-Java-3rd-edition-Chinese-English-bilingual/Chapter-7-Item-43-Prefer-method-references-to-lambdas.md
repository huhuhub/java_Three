## Chapter 7. Lambdas and Streams（λ表达式和流）

### Item 43: Prefer method references to lambdas

The primary advantage of lambdas over anonymous classes is that they are more succinct. Java provides a way to generate function objects even more succinct than lambdas: method references. Here is a code snippet from a program that maintains a map from arbitrary keys to Integer values. If the value is interpreted as a count of the number of instances of the key, then the program is a multiset implementation. The function of the code snippet is to associate the number 1 with the key if it is not in the map and to increment the associated value if the key is already present:

lambdas优于匿名类的主要优点是它们更简洁。 Java提供了一种生成函数对象的方法，它比lambdas：方法引用更简洁。这是一个程序的代码片段，它维护从任意键到Integer值的映射。如果该值被解释为密钥实例数的计数，则该程序是多集实现。代码段的功能是将数字1与密钥相关联（如果它不在映射中），并在密钥已存在时增加相关值：
```
map.merge(key, 1, (count, incr) -> count + incr);
```

Note that this code uses the merge method, which was added to the Map interface in Java 8. If no mapping is present for the given key, the method simply inserts the given value; if a mapping is already present, merge applies the given function to the current value and the given value and overwrites the current value with the result. This code represents a typical use case for the merge method.

请注意，此代码使用merge方法，该方法已添加到Java 8中的Map接口。如果给定键没有映射，则该方法只是插入给定值;如果已存在映射，则merge将给定函数应用于当前值和给定值，并使用结果覆盖当前值。此代码表示合并方法的典型用例。

The code reads nicely, but there’s still some boilerplate. The parameters count and incr don’t add much value, and they take up a fair amount of space. Really, all the lambda tells you is that the function returns the sum of its two arguments. As of Java 8, Integer (and all the other boxed numerical primitive types) provides a static method sum that does exactly the same thing. We can simply pass a reference to this method and get the same result with less visual clutter:


代码读得很好，但仍然有一些样板。参数count和incr不会增加太多值，并且占用相当大的空间。实际上，所有lambda告诉你的是该函数返回其两个参数的总和。从Java 8开始，Integer（以及所有其他盒装数字基元类型）提供了一个完全相同的静态方法求和。我们可以简单地传递对此方法的引用，并以较少的视觉混乱获得相同的结果：

```
map.merge(key, 1, Integer::sum);
```

The more parameters a method has, the more boilerplate you can eliminate with a method reference. In some lambdas, however, the parameter names you choose provide useful documentation, making the lambda more readable and maintainable than a method reference, even if the lambda is longer.

方法具有的参数越多，使用方法引用就可以消除的样板越多。但是，在某些lambdas中，您选择的参数名称提供了有用的文档，使得lambda比方法引用更易读和可维护，即使lambda更长。

There’s nothing you can do with a method reference that you can’t also do with a lambda (with one obscure exception—see JLS, 9.9-2 if you’re curious). That said, method references usually result in shorter, clearer code. They also give you an out if a lambda gets too long or complex: You can extract the code from the lambda into a new method and replace the lambda with a reference to that method. You can give the method a good name and document it to your heart’s content.

对于一个你不能用lambda做的方法引用，你无能为力（有一个模糊的例外 - 如果你很好奇，请参阅JLS，9.9-2）。也就是说，方法引用通常会导致更短，更清晰的代码。如果lambda变得太长或太复杂，它们也会给你一个out：你可以将lambda中的代码提取到一个新方法中，并用对该方法的引用替换lambda。您可以为该方法提供一个好名称，并将其记录在心脏的内容中。

If you’re programming with an IDE, it will offer to replace a lambda with a method reference wherever it can. You should usually, but not always, take the IDE up on the offer. Occasionally, a lambda will be more succinct than a method reference. This happens most often when the method is in the same class as the lambda. For example, consider this snippet, which is presumed to occur in a class named GoshThisClassNameIsHumongous:

如果您使用IDE进行编程，它将提供用方法引用替换lambda，只要它可以。您通常（但不总是）应该在优惠中使用IDE。偶尔，lambda将比方法引用更简洁。当方法与lambda属于同一类时，这种情况最常发生。例如，考虑这个片段，假定它出现在名为GoshThisClassNameIsHumongous的类中：

```
service.execute(GoshThisClassNameIsHumongous::action);
```

The lambda equivalent looks like this:

lambda等价物看起来像这样：

```
service.execute(() -> action());
```

The snippet using the method reference is neither shorter nor clearer than the snippet using the lambda, so prefer the latter. Along similar lines, the Function interface provides a generic static factory method to return the identity function, Function.identity(). It’s typically shorter and cleaner not to use this method but to code the equivalent lambda inline: x -> x.

使用方法引用的代码段既不比使用lambda的代码段更短也更清晰，所以更喜欢后者。类似地，Function接口提供了一个通用的静态工厂方法来返回Identity函数Function.identity（）。它通常更短更清洁，不使用此方法，而是编写等效的lambda内联：x  - > x。

Many method references refer to static methods, but there are four kinds that do not. Two of them are bound and unbound instance method references. In bound references, the receiving object is specified in the method reference. Bound references are similar in nature to static references: the function object takes the same arguments as the referenced method. In unbound references, the receiving object is specified when the function object is applied, via an additional parameter before the method’s declared parameters. Unbound references are often used as mapping and filter functions in stream pipelines (Item 45). Finally, there are two kinds of constructor references, for classes and arrays. Constructor references serve as factory objects. All five kinds of method references are summarized in the table below:

许多方法引用引用静态方法，但有四种方法引用不引用静态方法。其中两个是绑定和未绑定的实例方法引用。在绑定引用中，接收对象在方法引用中指定。绑定引用在本质上类似于静态引用：函数对象采用与引用方法相同的参数。在未绑定的引用中，在应用函数对象时，通过方法声明的参数之前的附加参数指定接收对象。未绑定引用通常用作流管道中的映射和过滤功能（第45项）。最后，对于类和数组，有两种构造函数引用。构造函数引用充当工厂对象。所有五种方法参考总结在下表中：

|    Method Ref Type    |       Example       |      Lambda Equivalent     |
|:-------:|:-------:|:-------:|
|   Static  |     Integer::parseInt    |   str ->   |
|   Bound  |     Instant.now()::isAfter    |   Instant then =Instant.now(); t ->then.isAfter(t)   |
|   Unbound  |     String::toLowerCase    |   str ->str.toLowerCase()   |
|   Class Constructor  |     TreeMap<K,V>::new    |   () -> new TreeMap<K,V>   |
|   Array Constructor  |     int[]::new    |   len -> new int[len]   |

In summary, method references often provide a more succinct alternative to lambdas. **Where method references are shorter and clearer, use them; where they aren’t, stick with lambdas.**  

总之，方法引用通常提供了一种更简洁的lambdas替代方法。 **如果方法参考更短更清晰，请使用它们;他们不在的地方，坚持使用lambdas。**