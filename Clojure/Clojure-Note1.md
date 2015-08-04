#Clojure Note-1

## 一、表达式表示法

1.	Clojure是一种Lisp方言，使用小括号"()"来定义代码范围。

2.	Clojure采用的是前缀表示法，运算符在前面。

举个栗子：
	
		(+ 1 2)
		
## 二、同像性

+ Clojure的代码就是Clojure自身数据结构来表示的。——“代码即数据”

## 三、Clojure Reader的职责

  虽然Clojure的编译和求值全部都是对Clojure的数据结构进行的，但是对于普通Clojure程序员来说，代码还是写成文本格式的。Clojure Reader会把我们写的普通文本格式的代码处理成Clojure的那些数据结构。
  
1. reader的所有操作是由一个叫做read的函数定义的，这个函数从一个字符流里读入代码的文本形式，产生这个文本形式对应的数据结构。
2. Clojure的REPL就是使用reader来读入文本代码的，read函数读出的每个完整的数据结构都会传给Clojure的运行时来求值。
3. reader的作用其实可以看作是一种反序列化机制。

举个栗子（;=表示函数的结果）

	(read-string "42")
	;=42
	
	(read-string "(+ 1 2)")
	;=(+ 1 2)
	
	;pr-str是将Clojure中的值默认打印到标准输出终端上，并且把这个值返回。
	(pr-str [1 2 3])
	;="[1 2 3]"
	
	(read-string "[1 2 3]")
	;=[1 2 3]
	
## 四、Clojure的基本数据类型

1.	字符串：Clojure中的字符串就是Java的字符串（java.lang.String的实例），表达形式也是一样的，双引号括起来。Clojure的字符串天然支持多行，不用使用任何特殊的语法形式。

2. 布尔值：true和false。

3. nil：表示未知的引用，和Java中的null一样。

4. 字符：Clojure中的字符是通过反斜杠加字符来表示的：

		(class \c)
		;=java.lang.Character
	而且，对于Unicode编码和octal编码，我们可以使用对应的前缀：
	
		\u00ff
		;=\y
		\041
		;=\!
   同时对于一些特殊字符，也有对应的常量：
   
	* \space
	* \newLine
	* \formfeed
	* \return
	* \backspace
	* \tab

## 五、关键字
   
  + 关键字从语法上来说，关键字始终以冒号开头；
  + 关键字求值（解析）为自身；
  + 关键字本身就是一个函数，它的作用就是查找它所对应的值。
  + 关键字常用作为键。
  
## 六、符号

  + 符号跟关键字一样，符号也是一种标识符，但不同的是，符号的值它所代表的Clojure运行时里面的那个值，这个值可以是var所持有的值（持有的可以是函数及其他值）、Java类、本地应用等。

  		(average [60 110 100])
  		;=90
  	这里的average就是一个符号。