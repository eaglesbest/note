# Clojure中集合

Clojure中的集合类型如下：

+ list
+ vector
+ set
+ map

同时Clojure还可以使用Java里面提供的所有集合类型，但是通常不会这样做，因为Clojure自带的集合类型更适合函数式编程。Clojure中的集合做了更高层的抽象，比如conj、count这些函数在任何集合中都可以使用，而不用理会具体是何种集合类型。

Clojure集合有着java集合所不具备的一些特性。所有的clojure集合是不可修改的、异源的以及持久的。不可修改的意味着一旦一个集合产生之后，你不能从集合里面删除一个元素，也往集合里面添加一个元素。异源的意味着一个集合里面可以装进任何东西（而不必须要这些东西的类型一样）。持久的以为着当一个集合新的版本产生之后，旧的版本还是在的。CLojure以一种非常高效的，共享内存的方式来实现这个的。比如有一个map里面有一千个name-valuea pair, 现在要往map里面加一个，那么对于那些没有变化的元素， 新的map会共享旧的map的内存，而只需要添加一个新的元素所占用的内存。

要记住的是，因为clojure里面的集合是不可修改的，所以也就没有对集合进行修改的函数。相反clojure里面提供了一些函数来从一个已有的集合来高效地创建新的集合 — 使用[persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure)。同时也有一些函数操作一个已有的集合（比如vector)来产生另外一种类型的集合(比如LazySeq), 这些函数有不同的特性。

## 通用的函数

### count

count返回集合里面元素的个数：

	user=> (count [19 "yellow",true])
	3

### conj

conj是conjoin的缩写。其功能是在一个集合加入一个元素，组成一个新的集合返回。

	user=> (conj [1 2 3] 4)
	[1 2 3 4]

注意：conj并不会改变原来的集合，只是用一个创建一个新的集合，将原来集合和新添加的元素一起组合一个新的集合。举个例子：

	user=> (do
         (def v [1 2 3])
         (conj v 4))
	[1 2 3 4]
	user=> (print v)
	[1 2 3]nil
	
### reverse

reverse把函数里面的元素反转。

	user=> (reverse [1 2 3 4])
	(4 3 2 1)

### map

map 对一个给定的集合里面的每一个元素调用一个指定的方法，然后这些方法的所有返回值构成一个新的集合（LazySeq）返回。这个指定了函数也可以有多个参数，那么你就需要给map多个集合了。如果这些给的集合的个数不一样，那么执行这个函数的次数取决于个数最少的集合的长度。比如：

	user=> (map #(+ % 3) [1 2 3])
	(4 5 6)
	user=> (map + [2 4 7] [5 6] [1 2 3 4])
	(8 12)
	
### filter

对于集合中的每一个元素调用指定的方法，然后返回符合指定函数的元素构成的一个新的集合。

	;;返回集合中字符超过3个元素
	user=> (filter #(> (count %) 3) ["Moe" "Larry" "Curly" "Shemp"])
	("Larry" "Curly" "Shemp")
	
### apply

apply 把给定的集合里面的所有元素一次性地给指定的函数作为参数调用，然后返回这个函数的返回值。所以apply与map的区别就是map返回的还是一个集合，而apply返回的是一个元素， 可以把apply看作是SQL里面的聚合函数。比如：

	user=> (apply + [2 4 7])
	13
	
### 从集合中获取一个元素的函数

+ **first**

从集合中返回第一个元素

	user=> (def stooges ["Moe" "Larry" "Curly" "Shemp"])
	#'user/stooges
	user=> (first stooges)
	"Moe"
	
+ **second**

从集合中返回第二个元素

	user=> (second stooges)
	"Larry"
	
+ **last**

从集合中返回最后一个元素

	user=> (last stooges)
	"Shemp"
	
+ **nth**

从集合中返回指定索引（index）的元素

	user=> (nth stooges 2) ;;索引从0开始
	"Curly"
	
### 从集合中获取多个元素的函数

+ next

返回除第一个的剩余的元素。

	user=> (next stooges)
	("Larry" "Curly" "Shemp")
	
+ butlast

返回除最后一个元素外的所有元素

	user=> (butlast stooges)
	("Moe" "Larry" "Curly")
	
+ drop-last

返回除最后n个元素的集合

	user=> (drop-last 1 stooges)
	("Moe" "Larry" "Curly")
	user=> (drop-last 2 stooges)
	("Moe" "Larry")
	
+ nthnext

返回从指定索引位置开始到最后的所有元素

	user=> (nthnext stooges 1)
	("Larry" "Curly" "Shemp")
	
### 谓词函数

有一些谓词函数测试集合里面每一个元素然后返回一个布尔值，这些函数都是”short-circuit”的，一旦它们的返回值能确定它们就不再继续测试剩下的元素了，有点像java的&&和or, 
	
+ every?

判断集合中每一个元素是否符合指定函数。如判断stooges是否每个元素都是字符串类型：

	user=> (every? #(instance? String %) stooges)
	true
	
+ not-every?

见名会意。这个函数和every?刚好相反，用于判断集合中是否存在不满足指定函数的元素。
如判断判断下面集合是否每个元素都是字符串类型：

	user=> (not-every? #(instance? String %) ["Clojure","Java","Erlang","Haskell"])
	false
	user=> (not-every? #(instance? String %) ["Clojure","Java",1])
	true
	
+ some

判断集合中是否有元素符合指定函数。如果有返回true，否则返回nil

	user=> ;;判断stooges中是否有数字类型
	user=> (some #(instance? Number %) stooges)
	nil
	user=> (some #(instance? String %) stooges)
	true

+ not-any?

判断集合中元素是全部都不满足指定函数。

	user=> ;;判断是否stooges中元素都是都不是数字类型
	user=> (not-any? #(instance? Number %) stooges)
	true

	user=> (not-any? #(instance? Number %) (conj stooges 1))
	false
	
----

## 数据结构

### Lists

Lists是一个有序的元素的集合 — 相当于java里面的LinkedList。这种集合对于那种一直要往最前面加一个元素，干掉最前面一个元素是非常高效的(O(1)) — 想到于java里面的堆栈, 但是没有高效的方法来获取第N个元素， 也没有高效的办法来修改第N个元素。

下面是创建同样的list的多种不同的方法：

	user=> (def stooges (list "Moe" "Larry" "Curly"))
	#'user/stooges
	user=> (def stooges (quote ("Moe" "Larry" "Curly")))
	#'user/stooges
	user=> (def stooges '("Moe" "Larry" "Curly"))
	#'user/stooges
	
some 

可以用来检测一个集合是否含有某个元素. 它的参数包括一个谓词函数以及一个集合。你可以能会想了，为了要看一个list到底有没有某个元素为什么要指定一个谓词函数呢？其实我们是故意这么做来让你尽量不要这么用的。从一个list里面搜索一个元素是线性的操作（不高效），而要从一个set里面搜索一个元素就容易也高效多了，看下面的例子对比：

	user=>(some #(= % "Moe") stooges)
	true
	user=>(some #(= % "Mark") stooges)
	nil
	user=>; Another approach is to create a set from the list
	user=>; and then use the contains? function on the set as follows.
	user=>(contains? (set stooges) "Moe") 
	true
	
conj 的完整词义是 conjoin ， 表示『相连接』的意思， 它用于将元素和 collection 拼接起来。
需要注意的是， 根据 coll 的类型， 组合会发生在 coll 的不同地方， 也即是， 元素 x 可能会被加入到 coll 的最左边，也可能会被加入到最右边。

	user=> (conj stooges "Shemp")
	("Shemp" "Moe" "Larry" "Curly")


cons (cons x seq)是返回一个新的序列， 序列的第一个元素是 x ， 而 seq 则是序列的其余部分。

	user=> (cons "Shemp" stooges)
	("Shemp" "Moe" "Larry" "Curly")

remove 函数创建一个只包含所指定的谓词函数测试结果为false的元素的集合:

	user=> (remove #(= % "Curly") more-stooges)
	("Shemp" "Moe" "Larry")
	
into 函数把两个list里面的元素合并成一个新的大list

	user=> (def kids-of-mike '("Greg" "Peter" "Bobby"))
	#'user/kids-of-mike
	user=> (def kids-of-carol '("Marcia" "Jan" "Cindy"))
	#'user/kids-of-carol
	user=> (def brady-bunch (into kids-of-mike kids-of-carol))
	#'user/brady-bunch
	user=> brady-bunch
	("Cindy" "Jan" "Marcia" "Greg" "Peter" "Bobby")
	
peek 和pop 可以用来把list当作一个堆栈来操作. 她们操作的都是list的第一个元素。

	






