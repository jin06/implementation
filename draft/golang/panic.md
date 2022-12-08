###

panic逻辑主要在 runtime/panic.go中

### gorecover

在 1008行，主要逻辑

- recover必须作为eferred的来运行，即关键字 defer 定义的函数。并且要所有defer的最上边，也就是最后运行的defer
- 先调用 getg() 获取g
- 在调用g的 _panic , 如果定义了recover，则返回_panic中的  p.arg 
- p.arg 的定义为 panic的argument


### deferproc 

- 270行
- 实现语法中defer func()功能。
- go编译时会将相应代码编译成这个函数
- 函数执行的过程
    - 判断当前g是否在系统调用栈上，如果在就不执行defer并跑出错误
    - 生成一个defer对象
    - defer插入到 g的defer链中，当前的defer在最外层（新加入的defer在最外层，和defer的执行顺序一致）
    - 注意 deferproc(fn func()) 中传入的参数fn就是defer 对应的函数。传入的fn是一个没有任何参数的函数，所以go中的defer调用的有参数的函数时，需要在加入到defer链时就已经计算好了

### deferprocStack 

是对defer的优化，deferproc本身是将defer对应的func分配到了堆上，在生成时涉及到defer func从栈复制到堆，而执行又要将堆复制到栈。

deferprocStack 的逻辑是将defer func作为当前函数的 一个局部变量，直接放到了栈上。

有一些情况，例如循环或者是goto引起的一些不确定性的引用，那么此时就不能使用deferprocStack。
总之就是如果能够确定有多少defer func 那么这些是可以使用deferprocStack，否则不能使用

这些都是编译器在编译时确定的。


### open-coded

这个是对deferprocStack的优化，可以解决例如 if 判断语句下的defer func的执行。

go采用了一个自己来表示8个defer func 是否执行，所以open-coded最大支持 函数中有8个以内的 defer func

编译器在优化的时候，会将defer func转成普通的func 根据响应defer func的顺序放在整个函数的最后，并加入原有判断条件和响应的defer是否执行的判断，并且原有判断条件的地方加入判断defer的条件的代码

### deferreturn 

448行 

- 执行defer function的逻辑
- 编译器会在任何调用defer的函数的位置插入这个调用
- 整个defer的执行时遍历所有的defer，一个一个执行。
- 其中有一个优化，就是如果开启了openDefer ， 则执行逻辑会变化，没有开启，就正常执行defer 后的函数
    - 开启openDefer后，会执行 runOpenDeferFrame(d *_defer) bool 

### gopanic 

执行panic() 的逻辑，go编译时会将panic() 替换成这个函数，。

804行

- 执行已经进入到defer调用链路中的func

