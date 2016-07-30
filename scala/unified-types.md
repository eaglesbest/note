##统一类型

对比Java，在Scala中所有的值都是对象（包括数值和函数）。因为Scala是基于类的，所以所有的值都是类的实例。下面的图说明类的层级关系。

![Scala class hierarchy](../charts/classhierarchy.img_assist_custom.png)

### Scala类的层级关系

`scala.Any`是所有类的超类（父类）。它有两个直接的子类`scala。AnyVal`和`scala.AnyRef`，它们代表两类不同的世界：值类和引用类。所有的值类是被预定义的，它们符合类似Java语言的原语类型；所有的其他类定义为引用类型。默认情况下，用户自己定义的类为引用类型，所以它们总是（间接）`scala.AnyRef`的子类。

在Scala中，每一个用户定义的类隐式继承了`scala.ScalaObject`特质（trait）。类在Scala运行基础（比如Java运行环境）不扩展`scala.ScalaObject`。

如果Scala是用于java运行环境的上下文，然后`scala.anyref`对应`java.lang.Object`。请注意，上面的图表也显示了值类之间的隐式转换。

这里的这个这例子演示了数字、字符、布尔值与函数都可以像对象一样。

```
object UnifiedTypes extends App {
  val set = new scala.collection.mutable.LinkedHashSet[Any]
  set += "This is a string"  // add a string
  set += 732                 // add a number
  set += 'c'                 // add a character
  set += true                // add a boolean value
  set += main _              // add the main function
  val iter: Iterator[Any] = set.iterator
  while (iter.hasNext) {
    println(iter.next.toString())
  }
}
```