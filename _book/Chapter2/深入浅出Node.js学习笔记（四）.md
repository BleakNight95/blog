# 深入浅出Node.js学习笔记（四）

## 异步编程

Node是首个将异步大规模带到应用层面的平台，它从内在运行机制到API的设计，无不透露出异步的气息来。

## 1.函数式编程

在JavaScript中，函数作为一等公民，使用上非常自由，无论调用它，或者作为参数，或者作为返回值均可。

函数式编程是JavaScript异步编程的基础。

### 1.1高阶函数 

高阶函数把函数作为参数，或是将函数作为返回值的函数。

```
function Night(x){
	return function (){
		return x;
	}
}
```

事件的处理方式正是基于高阶函数的特性的灵活性来完成的。

ECMAScript5提供的高阶函数：

1. forEach();
2. map();
3. reduce();
4. reduceRight();
5. filter();
6. every();
7. some();

### 1.2偏函数用法

偏函数用法是指创建一个调用另外一个部分-参数或者变量已经预置的函数-的函数的用法。

例子：

```
var toString = Object.prototype.toString;
var isString = function (obj) {
	return toString.call(obj) == '[object string]';
};
var isFunction = function (obj) {
	return toString.call(obj) == '[object function]';
};
```

例子改进(工厂模式)：

```
var isType = function (type) {
	return function (obj) {
		retirn toString.call(obj) == '[object' + type + ']';
	}
};
var isString = isType('String');
var isFunvtion = isType('Funvtion');
```

这种通过指定部分参数来产生一个新的定制的形式就是偏函数。

## 2.异步编程的优势与难点

### 2.1优势

Node带来的最大特性是基于事件驱动的非阻塞I/O模型。非阻塞I/O可以使CPU与I/O并不相互依赖等待，让资源更好的利用。对于网络应用而言，并行带来的想象空间巨大。延展而开是分布式和云。

由于事件循环模型需要应对海量的请求，海量请求同时作用在单线程上，就需要防止任何一个计算耗费过多的CPU时间片。至于是计算密集型，还是I/O密集型，只要计算不要影响异步I/O的调度，那就不构成问题。

### 2.2难点

难点：

1. 异常处理；

   编写异步方法遵循的原则：

   - 必须执行调用者传入的回调函数；
   - 正常传递异常供调用者判断；

2. 函数嵌套过深；

3. 阻塞代码；

4. 多线程编程；

   使用Web Workers,利用消息机制是合理使用多核CPU的理想模型。

   Web Workers能够解决利用CPU和减少阻塞UI渲染，但是不能解决UI渲染的效率问题。

5. 异步转同步；

## 3.异步编程解决方案

### 3.1事件发布/订阅模式

事件监听器模式是一种广泛用于异步编程的模式，是回调函数的事件化，又称发布/订阅模式。

事件发布/订阅模式：

```
// 订阅
emitter.on("event1",function(message){
	console(message);
})
// 发布
emitter.emit("event1","I am message!");
```

订阅事件就是个高阶函数的应用。事件发布/订阅模式可以实现一个事件与多个回调函数的关联，这些回调函数又称为事件侦听器。

通过emit()发布事件后，消息会立即传递给当前事件的所有侦听器执行。侦听器可以很灵活地添加和删除，使得事件和具体处理逻辑之间可以很轻松关联和解耦。

从另一种角度看，事件侦听器模式也是一种钩子(hook)机制，利用钩子导出内部数据或状态给外部的调用者。

Node对事件发布/订阅的机制的额外处理：

- 如果一个事件添加了超过10个侦听器，将会得到一个警告；

- 为了处理异常，EventEmitter对象对error事件进行了特殊对待；

1. 继承events模块

Node中Stream对象继承EventEmitter的例子：

```
var events = require('events');
function Stream (){
	events.EventEmitter.call(this);
}
util.inherits(Stream,events.EventEmitter);
```

2. 利用事件队列解决雪崩问题

once():侦听器只能执行一次，在执行之后就会将它与事件的关联移除。

采用once()解决雪崩问题。

雪崩问题：

在高访问量、大并发量的情况下缓存失效的情景，此时大量的请求同时涌入数据库中，数据库无法同时承受如此大的查询请求，进而往前影响网站的整体的响应速度。

一条数据库查询语句的调用：

```
var select = function (callback) {
	db.select("SQL",function (results) {
		callback(results);
	});
};
```

3. 多异步之间的协作方案

一般而言，事件与侦听器的关系是一对多，但在异步编程中，也会出现事件与侦听器的关系是多对一的情况，也就是说一个业务逻辑可能依赖两个通过回调或事件传递的结果。回调嵌套过深的原因就是如此。

由于多个异步场景中回调函数的执行并不能保证顺序，且回调函数之间没有任何交集，所以需要借助一个第三方函数和第三方变量来处理异步协作的结果。这个用于监测次数的变量叫做**哨兵变量**。

4. EventProxy的原理

   EventProxy来自于Backbone的事件模块，Backbone的事件模块是Model、View模块的基础功能。

   EventProxy将all当做一个事件流的拦截层，在其中注入一些业务来处理单一事件无法解决的异步处理问题。

5. EventProxy的异常处理

   EventProxy提供了fail()和done()这两个实例方法来优化异常处理，使得开发者将精力关注在业务实现，而不是在异常捕获上。

### 3.2Promise/Deferred模式

使用事件的方式时，执行流程需要被预先设定。即便是分支，也需要预先设定，这是由发布/订阅模式的运行机制所决定的。

是否有一种先执行异步调用，延迟传递处理的方式呢？

答案是Promise/Deferred模式。

1. **Promises/A**
   - Promise操作只会处在：未完成态、完成态、和失败态的一种；
   - Promise的状态只能从未完成态向完成态或失败态转化，不能转化。完成态和失败态不能相互转化；
   - Promise的状态一旦转化，将不能被更改；

一个Promise对象只要具备then()方法即可。

对于then()方法的要求：

- 接收完成态、错误态的回调方法。在操作完成或者出现错误的时候，将会调用对应的方法；
- 可选地支持progress事件回调作为第三个方法；
- then()方法只接受function对象，其余对象将被忽略；
- then()方法返回的是Promise对象，支持链式调用；

then()方法所做的事情是将回调函数给存储起来，为了完成整个流程，还需要触发执行这些回调函数的地方，实现这些功能的对象通常被称为Deferred,即延迟对象。

Promise和Dererred的差别：

Promise作用于外部，通过then()方法暴露给外部已添加自定义逻辑；

Dererred作用于内部，用于维护异步模型的状态；

Promise是高级接口，事件是低级 接口。低级接口可以构建更多更复杂的场景，高级接口一旦定义。不太容易变化，不再有低级接口的灵活性。但对于解决典型问题非常有效。

Promise通过封装异步调用，实现了正向用例和反向用例的分离以及逻辑处理延迟。

Promise需要封装，但是强大，具备很强的侵入性，纯粹的函数则较为轻量，但功能相对较弱。

2. **Promise中的多异步协作**

   类似于EventProxy。

3. **Promise的进阶知识**

   在API的暴露上，Promise模式比原始的事件侦听和触发略为优美，缺陷是需要为不同的场景封装不同的API，没有直接的原生事件那么灵活。

   Promise的秘诀其实在于对队列的操作。

   - 支持序列执行的Promise

     理想的编程体验是前一个的调用的结果作为下一个调用的开始，就是所谓的链式调用。

     要让Promise支持链式执行的步骤：

     1. 将所有的回调都存在队列中；
     2. Promise完成时，逐个执行回调，一旦监测到返回了新的Promise对象，停止执行，然后将当前的Deferred对象的promise引用改变为新的Promise对象，并将队列中余下的回调转交给他。

   - 将APIPromise化

### 3.3流程控制库

1. 尾触发与Next

   尾触发：

   需要手工调用才能持续执行后续调用的。

   常见的关键词是Next.

   尾触发目前应用最多的地方是Connect的中间件。

   中间件最简单的例子：

   ```
   function (req,res,next) {
   	//中间件
   }
   ```

   每个中间件传递请求对象、响应对象和尾触发函数，通过队列形成一个处理流。

   中间件机制使得在 处理网络请求时，可以像面向切面编程一样进行过滤、验证、日志等功能，而不与具体业务逻辑产生关联，以致产生耦合。

   原始的next()方法较为复杂，简化和的原理十分简单，取出队列的中间件并执行，同时传入当前方法以实现递归调用，达到持续触发的目的。

2. async

   - 异步的串行执行
   - 异步的并行执行
   - 异步调用的依赖处理
   - 自动依赖处理

3. Step

   比async更轻量。

   Step接收任意数量的任务，所有的任务都将会串行依次执行。

   Step与事件模式、Promise、async都不同的一点在于Step用到了this关键字。事实上，它是Step内部的一个next()方法，将异步的调用的结果传递给下一个任务作为参数，并调用执行。

   - 并行任务执行
   - 结果分组

4. wind

   wind为JavaScript语言提供了一个monadic扩展，能够显著提高一些常见场景下的异步编程体验。

   - 异步任务定义
   - $await()与任务模型
   - 异步方法转换辅助函数

## 4.异步并发控制

异步I/O与同步I/O的显著差距：

- 同步I/O因为每个I/O都是彼此阻塞的，在循坏体中，总是一个接着一个调用，不会出现耗用文件描述符太多的情况，同时性能也是低下的；

- 异步I/O,虽然并发容易实现，但是由于太容易实现，依然需要控制；

### 4.1 bagpipe的解决方案

bagpipe模块的解决思路：

- 通过一个队列来控制并发量；
- 如果当前活跃(指调用发起但未执行回调)的异步调用量小于限定值，从队列中取出执行；
- 如果活跃调用达到限定值，从队列中取出新的异步调用执行；
- 每个异步调用结束时，从队列中取出新的异步调用执行；

bagpipe类似于打开了一道窗口，允许异步调用并行进行，但是严格限定上限。仅仅在调用push()时分开传递，并不对原有API有任何侵入。

- 拒绝模式
- 超时控制

### 4.2 async的解决方案

async提供的处理异步调用的限制：parallelLimit()。