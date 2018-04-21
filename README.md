# Java基础
## JVM原理
JVM本身是介于JAVA编译器和操作系统之间的程序，这个程序提供了一个无视操作系统和硬件平台的运行环境

### 内存分配

![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j1.jpg)

所有线程共享的数据区： 

1.	方法区: 存储已被虚拟机加载的类信息、静态变量、编译后代码等数据。并使用永久代来实现方法区，1.8后被元空间替代，元空间并不在虚拟机中，而是使用本地内存，要画到上图那就是图外了。
2.	堆区: 我们常说用于存放对象的区域，1.7之后字符串常量池移到这里。

每个线程私有的数据区： 

1.	虚拟机栈: 方法执行时创建一个栈帧，用于存储局部变量、操作数栈、动态链接、方法出口等信息。每个方法一个栈帧，互不干扰。
2.	本地方法栈: 用于存放执行native方法的运行数据。
3.	程序计数器: 当前线程所执行的字节码的指示器，通过改变计数器来选取下一条需要执行的字节码指令。

直接内存：

* 直接内存并非Java标准。
* JDK1.4 加入了新的 NIO 机制，目的是防止 Java 堆 和 Native 堆之间往复的数据复制带来的性能损耗，此后 NIO 可以使用 Native 的方式直接在 Native 堆分配内存。
* 直接内存区域是全局共享的内存区域。

### Java对象不都是分配在堆上
废话，还有线程使用的虚拟机栈上的啊，在方法体中声明的变量以及创建的对象，将直接从该线程所使用的栈中分配空间。
#### 逃逸分析
逃逸是指在某个方法之内创建的对象，除了在方法体之内被引用之外，还在方法体之外被其它变量引用到；这样带来的后果是在该方法执行完毕之后，该方法中创建的对象将无法被GC回收，由于其被其它变量引用。正常的方法调用中，方法体中创建的对象将在执行完毕之后，将回收其中创建的对象；故由于无法回收，即成为逃逸。

逃逸分析可以分析出某个对象是否永远只在某个方法、线程的范围内，并没有“逃逸”出这个范围，逃逸分析的一个结果就是对于某些未逃逸对象可以直接在栈上分配，由于该对象一定是局部的，所以栈上分配不会有问题。

#### TLAB
JVM在内存新生代Eden Space中开辟了一小块线程私有的区域TLAB（Thread-local allocation buffer）。在Java程序中很多对象都是小对象且用过即丢，它们不存在线程共享也适合被快速GC，所以对于小对象通常JVM会优先分配在TLAB上，并且TLAB上的分配由于是线程私有所以没有锁开销。因此在实践中分配多个小对象的效率通常比分配一个大对象的效率要高。   
也就是说，Java中每个线程都会有自己的缓冲区称作TLAB，在对象分配的时候不用锁住整个堆，而只需要在自己的缓冲区分配即可。

### 类加载机制
#### 初始化时机
new、静态字段或方法被使用、反射、父类、main函数调用

#### 加载过程

1. 加载（获取字节流并转换成运行时数据结构，然后生成Class对象）
2. 验证（验证字节流信息符合当前虚拟机的要求）
3. 准备（为类变量分配内存并设置初始值）
4. 解析（将常量池的符号引用替换为直接引用）
5. 初始化（执行类构造器-类变量赋值和静态块的过程）

#### 类加载器
启动类加载器：是虚拟机自身的一部分，它负责将 <JAVA_HOME>/lib路径下的核心类库  
扩展类加载器：它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库，开发者可以直接使用标准扩展类加载器  
系统类加载器：它负责加载系统类路径java -classpath或-D java.class.path 指定路径下的类库

如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是**双亲委派模式**。

采用双亲委派模式的是好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次。其次防止恶意覆盖Java核心API。


### 内存分配（堆上的内存分配）
![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j2.jpg)
#### 新生代
##### 进入条件
优先选择在新生代的Eden区被分配。
#### 老年代
##### 进入条件
1. 大对象，-XX:PretenureSizeThreshold 大于这个参数的对象直接在老年代分配，来避免新生代GC以及分配担保机制和Eden与Survivor之间的复制
2. 经过第一次Minor GC仍然存在，能被Survivor容纳，就会被移动到Survivor中，此时年龄为1，当年龄大于预设值就进入老年代  
3. 如果Survivor中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于等于该年龄的对象进入老年代  
4. 如果Survivor空间无法容纳新生代中Minor GC之后还存活的对象

### GC回收机制
#### 回收对象
不可达对象：通过一系列的GC Roots的对象作为起点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时则此对象是不可用的。  
GC Roots包括：虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈中JNI（Native方法）引用的对象。

彻底死亡条件：  
条件1：通过GC Roots作为起点的向下搜索形成引用链，没有搜到该对象，这是第一次标记。  
条件2：在finalize方法中没有逃脱回收（将自身被其他对象引用），这是第一次标记的清理。

#### 如何回收
新生代因为每次GC都有大批对象死去，只需要付出少量存活对象的复制成本且无碎片所以使用“复制算法”  
老年代因为存活率高、没有分配担保空间，所以使用“标记-清理”或者“标记-整理”算法

复制算法：将可用内存按容量划分为Eden、from survivor、to survivor，分配的时候使用Eden和一个survivor，Minor GC后将存活的对象复制到另一个survivor，然后将原来已使用的内存一次清理掉。这样没有内存碎片。  
标记-清除：首先标记出所有需要回收的对象，标记完成后统一回收被标记的对象。会产生大量碎片，导致无法分配大对象从而导致频繁GC。  
标记-整理：首先标记出所有需要回收的对象，让所有存活的对象向一端移动。

#### Minor GC条件
当Eden区空间不足以继续分配对象，发起Minor GC。

#### Full GC条件
1. 调用System.gc时，系统建议执行Full GC，但是不必然执行
2. 老年代空间不足（通过Minor GC后进入老年代的大小大于老年代的可用内存）
3. 方法区空间不足

## 集合
### List、Set、Map区别
Set中的对象不按特定方式排序，并且没有重复对象。但它的有些实现类能对集合中的对象按特定方式排序，例如TreeSet类，它可以按照默认排序，也可以通过实现java.util.Comparator<Type>接口来自定义排序方式。  
List中的对象按照索引位置排序，可以有重复对象，允许按照对象在集合中的索引位置检索对象，如通过list.get(i)方式来获得List集合中的元素。  
Map中的每一个元素包含一个键对象和值对象，它们成对出现。键对象不能重复，值对象可以重复。

### ArrayList与LinkedList区别

|ArrayList|LinkedList|
|--------|--------|
|数组|双向链表|
|增删的时候在扩容的时候慢，通过索引查询快，通过对象查索引慢|增删快，通过索引查询慢，通过对象查索引慢|
|扩容因子1.5倍|无|

### HashMap和HashTable区别

1. Hashtable中的方法是同步的，而HashMap中的方法在缺省情况下是非同步的。
2. Hashtable中，key和value都不允许出现null值。HashMap中，null可以作为键，这样的键只有一个，可以有一个或多个键所对应的值为null。
3. 哈希值的使用不同，HashTable直接使用对象的hashCode。而HashMap重新计算hash值。
4. Hashtable和HashMap它们两个内部实现方式的数组的初始大小和扩容的方式。

### ConcurrentHashMap原理
[http://www.jasongj.com/java/concurrenthashmap/](http://www.jasongj.com/java/concurrenthashmap/)

HashTable 在每次同步执行时都要锁住整个结构。ConcurrentHashMap 锁的方式是稍微细粒度的。 ConcurrentHashMap 将 hash 表分为 16 个桶（默认值）

#### Java7
![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j3.jpg)

ConcurrentHashMap 类中包含两个静态内部类 HashEntry 和 Segment。HashEntry 用来封装映射表的键 / 值对；Segment 用来充当锁的角色，每个 Segment 对象守护整个散列映射表的若干个桶。每个桶是由若干个 HashEntry 对象链接起来的链表。一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组。

#### Java8
![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j4.jpg)

Java 8为进一步提高并发性，摒弃了分段锁的方案，而是直接使用一个大的数组。同时为了提高哈希碰撞下的寻址性能，Java 8在链表长度超过一定阈值（8）时将链表（寻址时间复杂度为O(N)）转换为红黑树（寻址时间复杂度为O(long(N))

Java 8的ConcurrentHashMap同样是通过Key的哈希值与数组长度取模确定该Key在数组中的索引。

对于put操作，如果Key对应的数组元素为null，则通过CAS操作将其设置为当前值。如果Key对应的数组元素（也即链表表头或者树的根元素）不为null，则对该元素使用synchronized关键字申请锁，然后进行操作。如果该put操作使得当前链表长度超过一定阈值，则将该链表转换为树，从而提高寻址效率。

## 多线程
### 问：你怎么理解多线程的

1. 定义：多线程是指从软件或者硬件上实现多个线程并发执行的技术。具有多线程能力的计算机因有硬件支持而能够在同一时间执行多于一个线程，进而提升整体处理性能。
2. 存在的原因：因为单线程处理能力低。打个比方，一个人去搬砖与几个人去搬砖，一个人只能同时搬一车，但是几个人可以同时一起搬多个车。
3. 实现：在Java里如何实现线程，Thread、Runnable、Callable。
4. 问题：线程可以获得更大的吞吐量，但是开销很大，线程栈空间的大小、切换线程需要的时间，所以用到线程池进行重复利用，当线程使用完毕之后就放回线程池，避免创建与销毁的开销。

### 线程的生命周期
新建 -- 就绪 -- 运行 -- 阻塞 -- 死亡

### 多线程实现方案
1. 继承Thread类
2. 实现Runnable接口
3. 扩展一种：实现Callable接口。这个得和线程池结合

### 如何实现同步
[https://fangjian0423.github.io/2016/04/18/java-synchronize-way/](https://fangjian0423.github.io/2016/04/18/java-synchronize-way/)

### volatile
功能：

1. 主内存和工作内存，直接与主内存产生交互，进行读写操作，保证可见性；
2. 禁止 JVM 进行的指令重排序。

### ThreadLocal
使用`ThreadLocal<UserInfo> userInfo = new ThreadLocal<UserInfo>()`的方式，让每个线程内部都会维护一个ThreadLocalMap，里边包含若干了 Entry（K-V 键值对），每次存取都会先的都当前线程，然后得到该线程对象中的Map，然后与Map交互。

### 线程池
[ThreadPoolExecutor](https://www.jianshu.com/p/edd7cb4eafa0)

### 并发包工具类
ConcurrentHashMap
ThreadPoolExecutor

[https://blog.csdn.net/mzh1992/article/details/60957351](https://blog.csdn.net/mzh1992/article/details/60957351)

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

CyclicBarrier可以循环使用。比如，假设我们将计数器设置为10，那么凑齐第一批10个线程之后，计数器就会归零，然后接着凑齐下一批，10个线程。司令下达命令，需要召集10个士兵，然后分别执行10个任务，需要等到士兵集合完毕，才能下达具体的任务，需要10个任务都完成，才能宣布任务结束。

## NIO
一种同步非阻塞的高效IO交互模型，比同步阻塞IO区别如下：

1. 通过缓冲区而非流的方式进行数据的交互，流是进行直接的传输的没有对数据操作的余地，缓冲区提供了灵活的数据处理方式。
2. NIO是非阻塞的，意味着每个socket连接可以让底层操作系统帮我们完成而不需要每次开个线程去保持连接，使用的是selector监听所有channel的状态实现。
3. NIO提供直接内存复制方式，消除了JVM与操作系统之间读写内存的损耗。

### 同步异步、阻塞非阻塞
同步和异步在于多个任务执行过程中，后发起的任务是否必须等先发起的任务完成之后再进行，是方法被调用的顺序。

阻塞和非阻塞在于请求的方法是否立即返回，是方法被执行的方式。

# 数据库
## MySQL
### 引擎对比
1. InnoDB支持事务
2. InnoDB支持外键
3. InnoDB是聚集索引，数据文件是和索引绑在一起的，必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而MyISAM是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。
4. InnoDB有行级锁

因为MyISAM相对简单所以在效率上要优于InnoDB.如果系统读多，写少。对原子性要求低。那么MyISAM最好的选择。
如果系统读少，写多的时候，尤其是并发写入高的时候，还需要事务安全性。InnoDB就是首选了。

### SQL性能优化
* 对经常查询的列建立索引，为了避免全表扫描，建多了当数据改变时修改索引浪费资源
* 使用精确列名查询而不是*
* 减少嵌套查询
* 不用NOT IN,IS NULL,NOT IS NULL，无法使用索引

## 事务隔离级别
1. 原子性（Atomicity）：事务作为一个整体被执行 ，要么全部执行，要么全部不执行；
2. 一致性（Consistency）：保证数据库状态从一个一致状态转变为另一个一致状态；
3. 隔离性（Isolation）：多个事务并发执行时，一个事务的执行不应影响其他事务的执行；
4. 持久性（Durability）：一个事务一旦提交，对数据库的修改应该永久保存。

[https://www.jianshu.com/p/4e3edbedb9a8](https://www.jianshu.com/p/4e3edbedb9a8)

## 锁表、锁行
[https://segmentfault.com/a/1190000012773157](https://segmentfault.com/a/1190000012773157)

### 何时锁

### 悲观锁乐观锁、如何写对应的SQL
[https://www.jianshu.com/p/f5ff017db62a](https://www.jianshu.com/p/f5ff017db62a)

### 索引
#### 原理
我们拿出一本新华字典，它的目录实际上就是一种索引：非聚集索引。我们可以通过目录迅速定位我们要查的字。而字典的内容部分一般都是按照拼音排序的，这实际上又是一种索引：聚集索引。聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

主要使用[B+树](https://www.jianshu.com/p/3a1377883742)来构建索引，为什么不用红黑树是因为一般来说，索引本身较大，不会全部存储在内存中，会以索引文件的形式存储在磁盘上，然后是因为[局部性原理](https://www.cnblogs.com/xyxxs/p/4440187.html)，数据库系统巧妙利用了磁盘预读原理，将一个节点的大小设为等于一个页，这样每个节点只需要一次I/O就可以完全载入，(由于节点中有若干个数组，所以地址连续)。而红黑树这种结构，深度更深。由于逻辑上很近的节点（父子）物理上可能很远，无法利用局部性

#### 分析
好处：

1. 快速取数据
2. 唯一性索引能保证数据记录的唯一性
3. 实现表与表之间的参照完整性

缺点：

1. 索引需要占物理空间。
2. 当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，降低了数据的维护速度。

#### 使用场景
如果某个字段，或一组字段会出现在一个会被频繁调用的 WHERE 子句中，那么它们应该是被索引的，这样会更快的得到结果。为了避免意外的发生，需要恰当地使用唯一索引，并且我个人不推荐使用全文索引，尤其对于汉字来说，全文索引的开销太大了，得不偿失。

# 设计模式
## 模式分类举例
### 创建型
工厂模式、生成器模式
### 结构型
桥接模式、适配器模式、外观模式、组合模式
### 行为型
策略模式、迭代器模式、访问者模式、观察者模式、命令模式、中介者模式、状态模式
## 常用模式
## 单例模式
[http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)

枚举实现原理：枚举本质上是通过普通的类来实现的，只是编译器为我们进行了处理。每个枚举类型都继承自java.lang.Enum。每个枚举常量是一个静态常量字段，使用内部类实现，该内部类继承了枚举类。

静态代码块中初始化对象：懒汉  
静态类引用：线程安全
final方法：禁止序列化、克隆

## 管道-过滤器模式

[http://www.wangtianyi.top/blog/2017/10/08/shi-yao-shi-hou-neng-yong-shang-she-ji-mo-shi/](http://www.wangtianyi.top/blog/2017/10/08/shi-yao-shi-hou-neng-yong-shang-she-ji-mo-shi/)

## 装饰器模式
动态地将责任附加到对象上，如果要拓展功能，装饰器提供了比继承更有弹性的方式。

Java IO包中就使用了该模式，InputStream有太多的实现类如FileInputStream，如果要在每个实现类上加上几种功能如缓冲区读写功能Buffered，则会导致出现ileInputStreamBuffered, StringInputStreamBuffered等等，如果还要加个按行读写的功能，类会更多，代码重复度也太高。

所以使用FilterInputStream这个抽象装饰器来装饰InputStream，使得我们可以用BufferedInputStream来包装FileInputStream得到特定增强版InputStream，且增加装饰器种类也会更加灵活。

![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j9.png)

# Web框架
## Spring
### 什么是Spring
Spring是个包含一系列功能的合集，如快速开发的Spring Boot，支持微服务的Spring Cloud，支持认证与鉴权的Spring Security，Web框架Spring MVC。IOC与AOP依然是核心。

### Spring MVC流程

1. 发送请求——>DispatcherServlet拦截器拿到交给HandlerMapping
2. 依次调用配置的拦截器，最后找到配置好的业务代码Handler并执行业务方法
3. 包装成ModelAndView返回给ViewResolver解析器渲染页面

### 解决循环依赖
无参数构造器、字段注入

### Bean的生命周期

1. Spring对Bean进行实例化
2. Spring将值和Bean的引用注入进Bean对应的属性中
3. 容器通过Aware接口把容器信息注入Bean
4. BeanPostProcessor。进行进一步的构造，会在InitialzationBean前后执行对应方法，当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理
5. InitializingBean。这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。
6. DispostbleBean。Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁，如果Bean实现了接口，Spring将调用它的destory方法

### Bean的作用域

* singleton：单例模式，Spring IoC容器中只会存在一个共享的Bean实例，无论有多少个Bean引用它，始终指向同一对象。
* prototype：原型模式，每次通过Spring容器获取prototype定义的bean时，容器都将创建一个新的Bean实例，每个Bean实例都有自己的属性和状态。
* request：在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean，而且该bean仅在当前Http Request内有效。
* session：在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。
* global Session：在一个全局的Http Session中，容器会返回该Bean的同一个实例，仅在使用portlet context时有效。


### IOC（DI）
控制反转：原来是自己主动去new一个对象去用，现在是由容器工具配置文件创建实例让自己用，以前是自己去找妹子亲近，现在是有中介帮你找妹子，让你去挑选，说白了就是用面向接口编程和配置文件减少对象间的耦合，同时解决硬编码的问题（XML）

依赖注入：在运行过程中当你需要这个对象才给你实例化并注入其中，不需要管什么时候注入的，只需要写好成员变量和set方法

### Spring AOP
#### 介绍
面向切面的编程，是一种编程技术，是OOP（面向对象编程）的补充和完善。OOP的执行是一种从上往下的流程，并没有从左到右的关系。因此在OOP编程中，会有大量的重复代码。而AOP则是将这些与业务无关的重复代码抽取出来，然后再嵌入到业务代码当中。常见的应用有：权限管理、日志、事务管理等。

#### 实现方式
实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。Spring AOP实现用的是动态代理的方式。

#### Spring AOP使用的动态代理原理
jdk反射：通过反射机制生成代理类的字节码文件，调用具体方法前调用InvokeHandler来处理  
cglib工具：利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理

1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

# TCP
## TCP过程
## 为什么要第三次握手
[https://blog.csdn.net/qq_18425655/article/details/52163228](https://blog.csdn.net/qq_18425655/article/details/52163228)

# 数据结构
## 快速排序
![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j5.jpg)

首先任意选取一个数据（通常选用数组的第一个数）作为关键数据，然后将所有比它小的数都放到它前面，所有比它大的数都放到它后面，这个过程称为一趟快速排序。

不稳定，如果最小的值在最后面，那么在第一次排序就被放到第一位，后面有与该值相等的元素也不会交换。

最好情况：每次划分过程产生的区间大小都为n/2，一共需要划分log2n次，每次需要比较n-1次，O(nlog2n)  
最坏情况：每次划分过程产生的两个区间分别包含n-1个元素和1个元素，一共需要划分n-1次，每次最多交换n-1次，这就是冒泡排序了，O(n2)

# 高性能架构

## Redis
### 原理
单线程的IO复用模型。便于IO操作，不便于排序、聚合。
### 消息队列
Redis自带的PUB/SUB机制，即发布-订阅模式。这种模式生产者(producer)和消费者(consumer)是1-M的关系，即一条消息会被多个消费者消费，当只有一个消费者时即可以看做一个1-1的消息队列

或者PUSH/POP机制

但都没有可靠的重试机制，需要自己管理与备份

### 缓存击穿
查询一个数据库中不存在的数据，比如商品详情，查询一个不存在的ID，每次都会访问DB，如果有人恶意破坏，很可能直接对DB造成过大地压力。

当通过某一个key去查询数据的时候，如果对应在数据库中的数据都不存在，我们将此key对应的value设置为一个默认的值。
### 缓存失效
在高并发的环境下，如果此时key对应的缓存失效，此时有多个进程就会去同时去查询DB，然后再去同时设置缓存。这个时候如果这个key是系统中的热点key或者同时失效的数量比较多时，DB访问量会瞬间增大，造成过大的压力。

1. 将系统中key的缓存失效时间均匀地错开　　
2. 当我们通过key去查询数据时，首先查询缓存，如果此时缓存中查询不到，就通过分布式锁进行加锁
### 热点key
缓存中的某些Key(可能对应用与某个促销商品)对应的value存储在集群中一台机器，使得所有流量涌向同一机器，成为系统的瓶颈，该问题的挑战在于它无法通过增加机器容量来解决。

1. 客户端热点key缓存：将热点key对应value并缓存在客户端本地，并且设置一个失效时间。
2. 将热点key分散为多个子key，然后存储到缓存集群的不同机器上，这些子key对应的value都和热点key是一样的。

## JVM调优
[https://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html](https://www.ibm.com/developerworks/cn/java/j-lo-jvm-optimize-experience/index.html)

## 高并发怎么处理
问题：比较耗CPU的任务摆在这里，程序也无法提升性能了，该怎么办？

1. 先判断能否使用缓存，还是重新耗费CPU资源来创建一个
2. 用偏重CPU性能的机器来做这个功能的专项负载均衡
3. 请求太多的话，搞个消息队列排着，慢慢消费，同时前端提示需要一会才行

## 控制拿数据库连接池资源
问题：数据库连接池就那么几个，但是有很多来拿怎么办？加个超时限制怎么加？

## Lucene原理
### 倒排索引
不是由记录来确定属性值，而是由属性值来确定记录的位置。

#### 构建过程

1. 分词
2. Hash去重
3. 根据单词生成索引表，同时得到“词典文件”（词-> 单词ID）
4. 得到“倒排索引文件”（单词ID -> 包含文档ID的倒排列表）

#### 词典文件（HashMap）

![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j7.jpg)

#### 词典文件（B+树）

#### 倒排索引文件

![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j8.gif)

## 爬虫优化
### 下载工具
HttpURLConnection：本身的 API 不够友好，所提供的功能也有限  
HttpClient：功能强大  
OkHttp：是一个专注于性能和易用性的 HTTP 客户端。OkHttp 会使用连接池来复用连接以提高效率。OkHttp 提供了对 GZIP 的默认支持来降低传输内容的大小。OkHttp 也提供了对 HTTP 响应的缓存机制，可以避免不必要的网络请求。当网络出现问题时，OkHttp 会自动重试一个主机的多个 IP 地址。

### 抵抗反爬虫策略
#### 动态页面的加载
phantomjs + selenium + java

#### 如何抓取需要登录的页面
模拟登录之后将sessionId保存到request header的cookie中

#### 如何解决IP限制问题
买个支持ADSL的拨号服务器，便宜的一个月80，然后在上面搭建代理服务器，用爬虫连上去


#### 每分钟访问频率
寻找网站访问频率访问限制漏洞，自己探索不同网站的访问频率限制规则

#### 验证码
图像识别

### 存储查询大量的数据

因为下载下来的HTML文件都是小文件，所以使用HDFS存储。我们需要两种节点，namenode节点来管理元数据，datanode节点来存储设计数据。

![](https://github.com/xbox1994/2018-Java-Interview/raw/master/images/j6.jpg)

### 实现思路

* 文件被切块存储在多台服务器上
* HDFS提供一个统一的平台与客户端交互
* 每个文件都可以保存多个副本

### 优点
1. 每个数据的副本数量固定，直接增加一台机器就可以实现线性扩展
2. 有副本让存储可靠性高
3. 可以处理的吞吐量增大
