#Clojure并发编程


## delay

+ delay是一种让一些代码延迟执行的机制，代码只会在显示地调用deref的时候执行。

		(def d (delay (println "Running....")
		              :done))
		=> #'user/d
		(deref d)
		Running....
		=> :done

+ delay对于它所包含的代码只执行一次，然后把返回值缓存起来。后面所有使用deref对它进行的访问都是直接返回的，而不用去执行它所包含的代码。因此，也不会由于多次执行delay所包含的代码而导致副作用。这是delay的一个优势。
+ 多个线程同时对delay进行解引用，所有的线程都会被阻塞住直到delay所包含的代码被求值（只求值一次！），之后的所有线程就可以访问这个值了。
+ realized?可以用来检测delay是否已经获取到了值。它同样可以适用到future，promise。
	
		(realized? d)
		=> true


## future

+ Clojure的future会在另外一个线程里面返回执行它所包含的代码，对于future的调用会立即返回，它允许当前线程（比如你的REPL）可以继续执行。线程的结果会保存在future里面，可以通过解引用来获取这个值。
		
		(def long-calculation (future (apply + (range 1e8))))
		=> #'user/long-calculation
		@long-calculation
		=> 4999999950000000	
		
+ 和delay一样，如果future的执行还没有完成的时候去解引用的话，会阻塞当前的线程，因此下面的这个表达式会阻塞我们的REPL5秒钟。
+ 跟delay不一样的是，在解引用一个future的时候，你可以指定一个超时时间以及一个“超时值”，“超时值”会在解引用的时候返回。