# 深入浅出Node.js学习笔记（八）

## 构建Web应用

前后端采用的语言都是JavaScript,在跨越HTTP进行沟通时的好处;

- 无须切换语言环境，部分知识不会因为语言环境的切换而丢失，会有一些额外的好处；
- 数据(因为JSON)可以很好的实现跨前后端直接使用；
- 一些业务(如模板渲染)可以很自由地轻量地选择前端还是后端进行，因为编程语言相同，所以切换代价小；

## 1. 基础功能

对于一个Web应用而言，在具体的业务上，可能存在的需求有：

- 请求方法的判断；
- URL的路径解析；
- URL中查询字符串解析；
- Cookie的解析；
- Basic认证
- 表单数据的解析
- 任意格式文件的上传处理；
- Session(会话)；

### 1.1 请求方法

请求方法包括：

1. GET;
2. POST;
3. HEAD;
4. DELETE;
5. PUT;
6. CONNECT;

### 1.2 路径解析

客户端代理(浏览器)会将完整的URL地址解析成报文，将路径和查询部分放在报文第一行。

最常见的根据路径进行业务处理的应用是静态文件服务器，会根据路径去查找磁盘中的文件，然后将其响应给客户端。

还有一种常见的分发场景是根据路径来选择控制器，预设路径为控制器和行为组合，无须额外配置路由信息。

### 1.3 查询字符串

字符串会跟随在路径后，形成请求报文首行的第二部分。

在业务调用产生之前，中间件或者框架会将查询字符串转换，然后挂载在对象上供业务使用。

### 1.4 Cookie

1. 初识Cookie

   Cookie的处理分为如下几步：

   - 服务器向客户端发送Cookie；
   - 浏览器将Cookie保存；
   - 之后每次浏览器都会将Cookie发向服务器端；

   可选参数会影响浏览器在后续发送给服务器端的行为。主要为以下几个选项：

   - path;
   - Expires和Max-Age
   - HttpOnly
   - Secure

2. Cookie的性能影响

   针对Cookie设置过多，造成的带宽的浪费的性能优化方案：

   - 减少Cookie的大小

     如果在域名的根节点设置Cookie，几乎所有子路径下的请求都会带上这些Cookie。

   - 为静态组件使用不同的域名

     为不需要Cookie的组件换个域名可以实现减少无效Cookie的传输，还可以突破浏览器下载线程的限制，但是将会增加域名转换为IP的DNS查询。

   - 减少DNS查询

     减少DNS查询和使用不同的域名看似冲突，但使用DNS缓存会削弱这个副作用。

### 1.5 Session

通过Cookie，浏览器和服务器可以实现状态的记录。但Cookie并非完美，缺点是：体积过大；Cookie在前后端均可以被修改，数据极易被篡改和伪造。综上，Cookie对于敏感数据的保护是无效的。

为了解决Cookie敏感数据的问题，Session应运而生。

Session的数据只保留在服务端，客户端无法修改，数据的安全有一定的保障，数据也无需在协议中每次传输。

如何将每个客户的服务器的数据一一对应：

1. 基于Cookie来实现用户和数据的映射；

   虽然将所有的数据都放在Cookie中不可取，但是将口令放在Cookie中是可以的。口令一旦被篡改，就丢失了映射关系，也无法修改服务器端存在的数据了。

   Session的有效期通常较短，普遍的设置是20分钟，如果20分钟之内服务器端和客户端没有交互产生，服务器端将数据删除。由于数据过期时间较短，且在服务器端存储数据，因此安全性相对较高。

   口令是如何产生的？

   一旦服务器端启用了Session，它将约定一个键值作为Session的口令，这个值可以随意约定。一旦服务器端检查到用户请求Cookie没有携带该值，它就会为之生成一个值，这个值是唯一且不重复的值，并设定超时时间。

   这种方案依赖Cookie的实现。

   如果客户端禁止使用Cookie，这个世界大多数的网站将无法实现登录等操作。

2. 通过查询字符串来实现浏览器端和服务器端数据的对应；

   它的原理是检查请求的查询字符串，如果没有值，会先生成新的带值的URL。

3. 利用HTTP请求头中的ETag；

- Session与内存

  为了解决性能问题和Session数据无法跨进程共享的问题，常用的方案是将Session集中化，将原本分散在多进程里的数据，统一转移到集中的数据存储中。目前常用的工具是Redis、MEmcached等，通过这些高效的缓存，Node进程无须再内部维护数据对象，垃圾回收问题和内存限制问题都可以迎刃而解。

  采用第三方缓存来存储Session会引起的一个问题时引起网络访问。

  理论上来说，访问网络中的数据比访问本地磁盘中的数据要慢，涉及到握手、传输、网络终端自身的磁盘I/O等。

  采用第三方高速缓存的理由：

  - Node与缓存服务保持长连接，而非频繁的短连接，握手导致的延迟只影响初始化；
  - 高速缓存直接在内存中进行数据存储和访问；
  - 缓存服务通常与Node进程会比在相同的机器上或者相同的机房里，网络速度受到的影响较小；

- Session与安全

  Session的安全主要指如何让这个口令更加安全。

  一种做法是将这个口令通过私钥加密进行签名，使得伪造的成本较高。

  一种方案是将客户端的某些独有信息与口令作为原值，然后签名，这样攻击者一旦不在原始的客户端进行访问，就会导致签名失败。这些独有信息包括用户IP和用户代理(User Agent)。

- XSS漏洞

  XSS的全称是跨站脚本攻击(Cross Site Scripting)，通常都是网站开发者决定哪些脚本可以执行在浏览器端。

  XSS主要形成的原因多数是用户的输入没有被转义，而被直接执行。

### 1.6 缓存

在HTTP之上构建的应用，其客户端除了比普通桌面应用具备更轻量的升级和部署等特性外，在跨平台、跨浏览器、跨设备上也具有独特的优势。

传统客户端在安装后的应用过程中仅仅需要传输数据，Web应用还需要传输构成界面的组件(HTML、CSS、JavaScript)。

为了提高性能，关于缓存的规则：

- 添加Expires或Cache-Control到报文头中；
- 配置ETags；
- 让Ajax可缓存；

通常来说，POST、DELETE、PUT这类带行为性的请求操作一般不做任何缓存，大多数缓存只应用在GET请求中。

简单来说，本地没有文件时，浏览器会请求服务端的内容，并将这部分内容放置在本地的某个缓存目录中。第二次请求时，它将对本地文件进行检查，如果不能确定这份本地文件是否可以直接使用，它将发起一次请求。

所谓条件请求，就是在普通的GET请求报文中，附带If-Modified-Since字段。

它将询问服务器是否有更新的版本，本地文件的最后修改时间。如果服务器没有新的版本，只需响应一个304状态码，客户端就使用本地版本。如果服务器端有新的版本，就将新的内容发送给客户端，客户端放弃本地版本。

条件请求采用时间戳的方式实现的缺陷：

- 文件的时间戳改定但内容并不一定改定；
- 时间戳只能精确到秒级别，更新频繁的内容将无法生效；

HTTP1.1中引入ETag解决采用时间戳的缺陷。

ETag的全称是Entity Tag，由服务器端生成，服务器端可以决定它的生成规则。如果根据文件内容生成散列值，那么条件请求将不会受到时间戳改动造成的带宽浪费。

尽管条件请求可以在文件内容没有修改的情况下节省带宽，但是依然会发起一个HTTP请求，使得客户端依然会花一定时间来等待响应。最好的方案就是连条件请求都不用发起，在服务器端响应内容时，让浏览器明确地将内存缓存起来。在响应里设置Expires或Cache-Control头，浏览器将根据改值进行缓存。

Expires是一个GMT格式的时间字符串。

浏览器在接到这个过期值后，只有本地还存在这个缓存文件，在到期时间之前我它都不会发起请求。但是Expires的缺席在于浏览器和服务器之间的时间可能不一致，可能导致文件提前过期，或者到期后文件并没有被删除。

Cache-Control能够避免浏览器端与服务器端时间不同步带来的不一致问题，只要进行类似倒计时方式计算过期时间即可。Cache-Control的值还能设置public、private、no-cache、no-store等能够更精细地控制缓存的选项。

在浏览器中，Expires和Cache-Control同时存在的话，且被同时支持时，max-age会覆盖Expires。

- 清除缓存

  设置缓存可以达到节省带宽的目的，但是缓存一旦设定，当服务器端意外更新内容时，却无法通知客户端更新。这使得在使用缓存时，也要为其设定版本号，所幸浏览器是根据URL进行缓存。

  一般的更新机制有：

  - 每次发布，路径中跟随Web应用的版本号：http://love.bleaknight.top/?v=20200110
  - 每次发布，路径中跟随该文件内容的hash值：http://love.bleaknight.top/?hash=bleaknight

### 1.7 Basic认证

Basic认证是当客户端与服务器端进行请求时，允许通过用户名和密码实现的一种身份认证方式。

Basic认证有太多的缺点，虽然经过Base64加密后在网络中传输，但是这近乎明文，十分的危险，一般只有在HTTPS的情况才会使用。

## 2. 数据上传

在业务中，需要接受的一些数据包括：表单提交、文件提交、JSON上传、XML上传等。

### 2.1 表单数据

最常见的数据提交就是通过网页表单提交数据到服务器端。

###  2.2 其他格式

除了表单数据外，常见的提交还有JSON和XML文件等，都是依据Content-Type中的值决定，其中JSON类型的值为appliction/json，XML的值为application/xml。

### 2.3 附件上传

除了常见的表单和特殊格式的内容提交，还有一种比较独特的表单，特殊表单与普通表单的差异在于该表单中可以含有file类型的控件，以及需要指定表单属性enctype为multipart/form-data。

### 2.4 数据上传与安全

内存和CSRF相关的安全问题。

1. 内存限制

   在解析表单、JSON和XML部分，采用的策略是先保存用户提交的所有数据，然后在解析处理，最后才传递给业务逻辑。

   这种策略存在潜在的问题时，它仅仅适合数据量小的提交请求，一旦数据量过大，将发生内存被占光的情况。

   解决此问题的方案：

   - 限制上传内容的大小，一旦超过限制，停止接受数据，并响应400状态码；
   - 通过流式解析，将数据流导向磁盘中，Node只保留文件路径等小数据；

2. CSRF

   CSRF的全称是Cross-SIte Request Forgery(跨站请求伪造)。

## 3. 路由解析

### 3.1文件路径型

1. 静态文件

2. 动态文件

   在MVC模式流行起来之前，根据文件路径执行动态脚本也是基本的路由方式，处理原理是Web服务器根据URL路径找到对应的文件，如/index.asp或index.php。Web服务器根据文件名后缀去 寻找脚本的解释器，并传入HTTP请求的上下文。

   解析器执行脚本，并输出响应报文，达到完成服务的目的。

### 3.2 MVC

MVC模型的主要思想是将业务逻辑按职责分类，主要分为：

- 模型(Model)，数据相关的操作和封装；
- 视图(View)，视图的渲染；
- 控制器(Controller)，一组行为的集合；

最经典的分层模式：

![image-20200110161337011](C:\Users\zhoub\AppData\Roaming\Typora\typora-user-images\image-20200110161337011.png)

分层模式的工作模式的说明：

- 路由解析，根据URL寻找到对应的控制器和行为；
- 行为调用相关的模型，进行数据操作；
- 数据操作结束后，调用视图和相关数据进行页面渲染，输出到客户端；

如何根据URL做路由映射？

1. 通过手工关联映射，会有一个对应的路由文件来将URL映射到对应的控制器；
2. 通过自然关联映射，没有对应的路由文件；

1. **手工映射**

   手工映射除了需要手工配置路由外较为原始外，它对URL的要求十分灵活，几乎没有格式上的限制。

   - 正则匹配
   - 参数解析

2. **自然映射**

### 3.3 RESTful

RESTful的全称是Representational State Transfer(表现层状态转化)。

符合RESTful规范的设计，成为RESTful设计。其设计哲学主要将服务器提供的内容实体看作一个资源，并表现在URL上。

## 4. 中间件

中间件(middleware)来简化和隔离这些基础设施与业务逻辑之间的细节，让开发者能够关注在 业务的开发上，已达到提升开发效率的目的。

中间价的行为比较类似于Java过滤器(filter)的工作原理，就是在进入具体的业务处理之前，先让过滤器处理。

### 4.1 异常处理

使用next()方法添加err参数，并捕获中间件直接抛出的同步异常。

### 4.2 中间件与性能

中间件性能提升的点：

- 编写高效的中间件
- 合理利用路由，避免不必要的中间件执行

1. **编写高效的中间件**

   优化的方法：

   - 使用高效的方法。
   - 缓存需要重复计算的结果。
   - 避免不必要的计算。

2. **合理使用路由**

   合理的路由使得不必要的中间件不参与请求处理的过程。

## 5.页面渲染

### 5.1 内容响应

服务端响应的报文，最终都要被终端(命令行终端、代码终端、浏览器)处理。服务器端的响应从一定程度上决定或指示了客户端该如何处理响应的内容。

内容响应的过程中，响应报头中的Content-*字段十分重要。

1. MIME

   浏览器通过不同的Content-Type的值决定采用不同的渲染方式，这个值简称为MIME。

2.  附件下载

   在一些场景下，无论响应的内容是怎样的MIME值，需求中并不要求客户端去打开它，只需要弹出并下载它即可。为了满足这种需求，Content-Disposition字段应声登场。Content-Disposition字段影响的行为是客户端会根据它 的值是应该将报文数据当做即时浏览的内容，还是可下载的附件。当内容只需即时查看时，它的值为inline，当数据可以存为附件时，它的值为attachment。

3. 响应JSON

4. 响应跳转

### 5.2 视图渲染

主流的普通的HTML内容的响应总称为视图渲染。

在动态页面技术中，最终的视图是由模板和数据共同生成出来的。

模板是带有特殊标签的HTML片段，通过与数据的渲染，将数据填充到这些特殊标签中，最终生成普通的带数据的HTML片段。通常将这种渲染方法设计为render()，参数就是模板路径和数据。

### 5.3 模板

模板技术虽然多种多样，但它的实质就是将模板文件和数据通过模板引擎生成最终的HTML代码。

形成模板技术的4个要素：

- 模板语言
- 包含模板语言的模板文件
- 拥有动态数据的数据对象
- 模板引擎

模板和数据与最终结果相比，这里有一个静态、动态的划分过程，相同的模板和不同的数据可以得到不然的结果，不同的模板与相同的数据也能得到不同的结果。模板技术使得网页中的动态内容和静态内容变得不相互依赖，数据开发者与模板开发者只要约定好数据结构，两者就不用相互影响了。

模板技术实际上就是拼接字符串这样很底层的活，只是各种模板有着各自的优缺点和技巧。

1. 模板引擎

   使用render()方法实现一个简单的模板引擎。

   - 语法分解；
   - 处理表达式；
   - 生成待执行的语句；
   - 与数据一起执行，生成最终字符串；

   **模板编译**

   为了能够最终与数据一起执行生成字符串，需要将原始的模板字符串转换成一个函数对象。这个过程称为模板编译，生成的中间函数只与模板字符串相关，与具体的数据无关。如果每次都生成这个中间函数，就会浪费CPU。为了提升模板渲染的性能速度，通常会采用模板预编译的方式。

   通过预编译缓存模板编译后的结果，实际应用中就可以实现一次编译，多次执行，而原始的方式每次执行过程中都要进行一次编译和执行。

2. with的应用

   - 模板安全

     在使用模板技术的时候，与输入有关的变量一定要转义。

3. 模板逻辑

4. 集成文件系统

5. 字模板

   模板文件太大，太多复杂，会增加维护上的难度。有些模板是可以重用的，这催生了字模板(Partial View)的产生。字模板可以嵌套在别的模板中，多个模板可以嵌入同一个字模板中。

6. 布局视图

   字模板主要商务为了重用模板和降低模板的复杂度。字模板的另一种使用方式就是布局视图(layout),布局视图又称母版页，它与字模板的原理相同，但是场景稍有区别。

7. 缓存性能

   模板引擎的优化步骤：

   - 缓存模板文件
   - 缓存模板文件编译后的函数
   - 优化模板中的执行表达式

8. 小结

   模板技术的出现，将业务开发与HTML输出的工作分离开，它的设计原理就是单一职责原理。

### 5.4 Bigpipe

Bigpipe产生于Facebook公司的前端加载技术，它的提出主要是为了解决重数据页面的加载速度问题。

最终的HTML要在所有的数据获取完成后才输出到浏览器端。Node通过异步已经将多个数据源的获取并行起来了，最终的页面输出速度取决于两个数据请求中响应时间慢的那个。

BigPipe的解决思路是将页面分割成多个部分(pagelet)，先向用户输出没有数据的布局(框架)，将每个部分逐步输出到前端，再最终渲染填充框架，完成整个网页的渲染。

Bigpipe是一个需要前后端配合实现的优化技术：

- 页面布局框架(无数据的)；
- 后端持续性的数据输出；
- 前端渲染；

1. 页面布局框架

   页面布局架构由后端渲染而出。

2. 持续数据输出

   模板输出后，整个网页的渲染并没有结束，但用户已经可以看到整个页面的大体样子。

3. 前端渲染

4. 小结

   Bigpipe将网页布局和数据渲染分离，使得用户在视觉上觉得网页提前渲染好了，其随着数据输出的过程逐步渲染页面，使得用户能够感知页面是活的。



