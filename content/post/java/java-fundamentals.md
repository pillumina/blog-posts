---
title: "Java Fundamentals"
date: 2020-10-26T12:40:29+08:00
hero: /images/posts/java-dev.png
menu:
  sidebar:
    name: Java Fundamentals
    identifier: java-fundamentals
    parent: java odyssey
    weight: 10
draft: false
---
### JAVA对象的equals方法和hashCode方法是这样规定的

  1. 相等（相同）的对象必须有相等的哈希码

  2. 如果两个对象的哈希吗相同，它们不一定相同

### Java集合判断两个对象是否相等的规则：

  1. 判断两个对象的哈希码是否相等

  2. 判断两个对象用equals是否相等

所以重写其中一个方法，必须重写另一个方法

让对象可拷贝：

  1. 实现java.lang.Cloneable 2. 重写Object的clone()方法

### 由于GC的自动回收机制，并不能保证finalize方法被及时的执行，因为对象的回收时机具有不确定性，或者要么没有触发垃圾回收。这个方法被禁止调用，应该用显式的close()方法。

### Collection类的选择

Set: 

排序吗？

是： TreeSet / LinkedHashSet

否： HashSet

List

要安全吗？

是： Vector

否： ArrayList或者LinkedList --- 查询多ArrayList，增删多LinkedList

### Map常用子类：HashMap, HashTable, TreeMap, ConcurrentHashMap

HashMap: 非线程安全，性能高，基于数组和链表实现。

TreeMap：有序键值对，按key排序

HashTable: 线程安全, 性能低

ConcurrentHashMap: 线程安全且性能较好。Java1.7采取分段锁，1.8采用CAS+synchronized保证并发安全。

### LinkedHashMap

为HashMap的子类，内部还有一个双向链表维护键值对的顺序。支持插入顺序、访问顺序

* 插入顺序：先添加在前，后添加在后
* 访问顺序：即get/put操作，对一个键执行get/put操作后，对应的键值被移动到链表的末尾，所以最末尾的是最近访问的，最开始的是最久没有被访问的。

* 有5种构造方法，4个是插入顺序，只有一个按照指定访问顺序，可以用于实现LRUCache



### 集合初始化、大小和扩容

#### 建议在集合初始化时指定集合容量大小

如果没有设置，元素增加，resize表会重建hash，影响性能

InitialCapacity = （count / loader_factor (0.75)）+ 1 ，暂时无法确定初始值，可以设置为16

#### Java集合的默认大小和扩容

ArrayList、Vector: 初始为10

HashSet、HashMap: 初始为16

Vector：加载因子为1，扩容增量为原来容量的1倍

ArrayList: 扩容增量为原容量的0.5倍+1

HashSet：加载因子为0.75，扩容增量为原来容量的一倍

HashMap: 加载因子为0.75，扩容增量为原来容量的一倍



#### 初始化方法

```java
// asList得到的是长度和内容都固定的列表，无法修改
List<String> numbers = new ArrayList<>(Arrays.asList("1", "2", "3"));
// 一维数组
int[] a2 = new int[]{1, 2, 3}
// 二维数组
int[][] a4 = new int[4][4]
```

### WeakReference弱引用



### for-each循环优于传统for循环

for-each更简洁，也没有性能损失，在数组和list上均可使用（实现iterable的对象）

有三种情况不能用：

* 过滤删除指定元素
* 修改指定元素值
* 并行遍历多个集合



### 异常处理

Throwable、Error、Exception

Exception分为 Runtime Exception以及Checked Exception

* 运行时异常：编译能通过，程序不会处理运行时异常，程序会中止。 ArithmaticException, IllegalArgumentException，NullPointerException， IndexOutOfBoundsException...

* 受检查的异常：要么用try catch捕获，要么用throws声明抛出，交给父类处理，否则编译不会通过。 IOException，SQLException...



#### try catch finally 中的return

* 不管有没有异常，finally都会执行
* try catch有return时，finally仍然会执行
* finally是在return后面的表达式运算后执行的，没有返回运算的值，而是先把要返回的值保存起来，不管finally中的代码怎么样，返回的值都不会改变、所以函数的return值是在finally执行前确定的。
* finally中最好不要包含return，否则程序会提前退出，返回值不是try活着catch中保存的返回值。

- 核心: finally语句在try catch中的控制转移语句执行前执行(return, break, continue, throw)。不过也有点区别，return和throw把程序控制权转交给调用者，break和continue是在当前方法内转移。
- 由上述可知，finally语句如果里面有控制转移语句，有可能把try-catch里的return或者异常捕获给"吃了"，那么一些未处理的异常可能无法被抛出！



#### try-with-resources语句

任何实现java.lang.AutoCloseable接口的对象，和实现java.io.Closeable接口的对象，都可以当做资源使用。

在此语句中，也可以加catch和finally，但是任何catch和finally代码块都在所有被声明的资源被关闭后才会执行。

```java
try (Statement stmt = con.createStatement()) {
  .....
} catch (SQLException e){
  JDBCTutorialUtilities.printSQLExcepion(e);
}
```



#### 线程和线程池的异常处理

并发情况下，如果在父线程中启动了子线程，那么try catch finally不好捕获。

##### 线程的异常处理

* 子线程中try catch
* 为线程设置UncaughtExceptionHandler，具体可以用Thread.setUncaughtExceptionHandler设置当前线程的异常处理器（默认没有），Thread.setDefaultUncaughtExceptionHandler为整个程序设置默认的异常处理器。 **注意**：子线程中发生了异常，如果没有任何类来接手处理的话，是会直接退出的，不会记录任何日志。

##### 线程池的异常处理

* 通过Future的get方法捕获异常（推荐）
* 通过execute提交的任务，才能把它抛出的异常给UncaughtExceptionHandler
* 通过submit提交的任务，无论是抛出的未检测异常还是已经检查异常，都将被认为是任务返回状态的一部分，这个异常被Future.get封装在ExecutionException中重新抛出。



#### 异常泄漏敏感信息

敏感异常包装在非敏感异常抛出，不能防止敏感信息泄漏。比如FileNotFoundException被IOException包装，并没什么用。



### 反射

#### 如何获得Class对象

* Class.forName("类的全限定名")
* 实例对象.getClass()
* 类名.class (类字面常量) -- 此方法创建对Class对象的引用时，不会自动地初始化该Class对象，和forName方法不同
* 如果一个字段被static final修饰，称为编译时常量，所以不会对类进行初始化，在编译器把结果放入常量池。
* 一旦类被加载到了内存里，那么无论哪种方式获取该类的Class对象，返回的都是指向同一个java堆地址的引用。JVM不会创建两个相同类型的Class对象。
* 对于任意一个Class对象，都需要它的类加载器和这个类本身一同确定其在JVM中的唯一性。所以即使两个Class对象来源于同一个Class文件，只要类加载器不同，两个Class对象就不同

#### 基本数据类型的Class对象和包装类的Class对象不一样

```java
Class c1 = Integer.class;
Class c2 = int.class;
// not equals!
```



### 类私有变量的反射访问

需要注意的是，需要设置属性可达，否则会抛出IllegalAccessException

```java
Field[] fs = clazz.getDeclaredFields();
for (Field field: fs){
  // 设置属性可达
  field.setAccessible(true);
  // ...后续的打印访问
}
```



### 输入输出流

#### 字节流

* InputStream
  * FileInputStream 文件流，能处理二进制文件也能处理文本
  * BufferedInputStream 缓冲流，能处理二进制也能处理文本
* OutputStream
  * FileOutputStream 文件流， 能处理二进制文件也能处理文本
  * ..... 

#### 字符流

* Reader
  * FileReader 文件流，只能出来文本文件
  * BufferedReader 缓冲流，只能处理文本文件
* Writer
  * FileWriter
  * BufferedWriter

#### NIO

核心是Channel、Buffer、Selectors

* Channel，Java NIO数据的源头，Buffer的唯一接口，向缓冲区提供数据或者读取数据，双向读写，异步读写，FileChannel, DataChannel, SocketChannel, ServerSocketChannel等根据数据来源分类。比如文件IO，UDP，TCP网络的IO
* Buffer，Java NIO数据读写中转，数据缓存，适用于除了bool类型的所有数据类型，因为bool类型不能通过IO发送
* Selectors, 异步IO核心类，实现异步非阻塞IO操作，允许1个Selector线程管理&处理多个通道 Channel。不需要为每个channel去分配一个线程，也就是事件驱动而不是同步监视事件。

注意：Java普通的IO写操作不是线程安全的，java.nio.channels.FileChannel提供线程安全的写操作。

用NIO多线程往同一文件写入数据：

* 利用RandomAccessFile访问文件部分内容
* 利用FileChannel 对线程独占的文件块需要加锁
* 利用MappedByteBuffer对文件进行并发写入



### 线程

线程状态：New, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED

#### Thread类方法

run()中放逻辑，start()开始线程，直接吊run只会在本进程中执行

sleep()和wait()最大区别：

* Sleep()睡眠时，保持对象锁，仍然占有该锁
* wait()睡眠时，会释放对象锁，wait必须放在synchronized block中，否则会抛出IllegalMonitorStateException异常

interrupted()方法，本质只是设置该线程的中断标志，设置为true，并根据线程状态决定是否抛出异常，比如线程阻塞时就是抛出中断阻塞异常，标志位同时被清除。

join()等待该方法的线程执行完毕后再继续执行

调用一个线程的interrupt方法，分具体情况

- 如果线程处于被阻塞(sleep, wait, join)，那么线程立即退出被阻塞状态，抛出InterruptedException
- 如果线程处于正常活动状态，那么将这个线程的中断标志设置为true，仅此而已。被设置的线程依然继续正常运行。所以这个方法并不能真正中断线程。

wait(), notify(), notifyAll()都要在synchronized代码块中使用，因为会对对象的锁标志进行操作。

#### 线程的创建方式

1. 继承Thread，实现run方法，start调用
2. 实现Runnable接口， Thread(Runnable target, String name)
3. 实现Callable接口，和FutureTask共用

```java
Callable callable  = new MyCallable();
FutureTask<String> task = new FutureTask<String>(callable);
Thread thread1 = new Thread(task, 'thread1')
```



#### Runnable和Callable的区别

1. Callable的任务执行后可以返回值，而Runnable为void
2. call方法可以抛出异常而run不可以
3. call任务可以拿到future对象，表示异步计算的结果。
4. 加入线程池运行，run使用ExecutorService的execute方法，call使用submit方法



#### 线程退出方式

1. 执行完run
2. 用interrupt中断



#### 共享数据和synchronized

Runnable中默认是共享数据，需要增加synchronized关键字来设置互斥区，保护共享数据：

#### AtomicInteger

AtomicInteger，AtomicLong，AtomicLongArray，AtomicReference等原子类的类，主要用于高并发环境下的高效程序处理，帮助简化同步处理。++i和i++不是线程安全的，使用的时候不可避免要用synchronized，而AtomicInteger为线程安全的加减操作。



#### 线程同步机制

* 共享内存

  Java的实例域，静态域，和数组元素都是放在堆内存中的，是线程都可以访问到的，即共享。局部变量，方法定义参数，异常处理参数线程间不共享。出现线程安全问题的都是前者。Java内存模型（JMM）决定了一个线程对共享变量的写入何时对其他线程是可见的。

* 消息传递

  Exchanger可以在两个线程内交换数据，也只能是2个线程。A线程调用exchange方法，会进入阻塞，知道B线程也调用了exchange方法，而后以线程安全的方式交换数据，之后两个线程继续运行。Semaphore，可以控制同时访问的线程个数，用acquire获取许可，release释放许可。



#### Java线程池体系结构



### JDBC

#### DBCP连接池配置

1. 手动配置

   ```java
   BasicDataSource dataSource = new BasicDataSource();
   // 调用配置
   dataSource.setDriverClassName("com.mysql.jdbc.Driver");
   // 得到连接
   Connection conn = ds.getConnection();
   ```

2. 读取配置文件

   ```java
   BasicDataSourceFactory factory = new BasicDataSourceFactory;
   // 读取配置
   DataSource dataSource = factory.createDataSource(properties);
   // 得到连接
   Connection conn = ds.getConnection();
   ```

#### FetchSize设置每次缓存读取大小

oracle中每次默认读取10行，mysql驱动则是把整个结果全部读取到内存中才开始允许应用读取结果，所以mysql可能有OOM问题。 -----> 在每次执行SQL语句之前，设置ps.executeQuery()之前使用setFetchSize()函数设置大小。





### JAVA8

#### lambda

lambda表达式对值封闭，不对变量封闭。也就是局部变量在lambda表达式中如果要使用，必须声明final类型或者是隐式的final

```java
int num = 123;
Consumer<Integer> print = (s) -> System.out.println(num);
// 上述是可以的，因为num虽然没声明final，但是和final的变量表现一致
// 但是如果中间加个num ++;就不行了
```



#### Stream

处理集合的操作，可以执行非常复杂的查找，过滤和映射数据等操作。

1. 不是数据结构，不会保存数据
2. 不会修改原来数据源，只会把操作后的数据保存到另外一个对象（peek方法？）
3. 惰性求值，流在中间处理过程中，只是对操作进行了记录，不会立刻执行，等到执行终止操作的时候才会进行实际的计算。

* 中间操作
  * 无状态：元素的处理不受之前元素的影响  -- filter, map, peek, parallel
  * 有状态：该操作只有拿到所有元素之后才能进行下去 -- distinct, sorted, limit, skip, unordered
* 结束操作
  * 非短路操作： 必须处理所有元素才能得到最终结果 -- forEach, reduce, collect, max, min, coint，iterator
  * 短路操作：遇到某些符合条件的元素就可以得到最终结果，比如A or B -- anyMatch, allMatch, findAny, noneMatch



#### Optional

主要解决空指针异常



#### MetaSpace



### 泛型

#### 泛型类，接口，方法

泛型的类型参数只能是类类型，不能是简单类型

不能对确切的泛型类型使用instanceof操作, 下面是非法的

```java
if(ex_num instanceof Generic<Number>){}
```



#### 只在编译期有效



#### 泛型容器之间没有继承关系

```java
Plate<Fruit> p = new Plate<Apple>(new Apple());
// Type mismatch: cannot convert from Plate<Apple> to Plate<Fruit>
// apple is-a fruit
// plate with apple not-is-a plate with fruit
```

就算容器装的东西有继承关系，容器之间没有继承关系，需要泛型通配符



#### 泛型通配符

* 上界通配符 Plate<? extends Fruit>

  为泛型添加上边界，传入的类型实参必须是指定类型的子类型

  这个会使得往盘子里放东西的set()失效，get()还是有效的

* 下界通配符 Plate<? super Fruit>

  传入的类型实参必须是指定类型的父类型

  会使得get()方法部分失效，只能存放到Object对象里，而set()方法正常

* ?与T的区别

  * 对于编译器来说所有的T都代表同一个类型 比如

    ```java
    public <T> List<T> fill(T... t) 
    ```

    代表要么是String要么是Integer

  * 通配符只能用于填充泛型变量T，不能用于定义变量

* 频繁往外读取的，用上界extends，经常往里面插入的，用下界super



### 类的初始化过程

继承的子类： 静态 -- 父类 -- 子类

interface中可以有静态方法，不能有普通方法，普通方法需要用defult加默认实现

interface中的变量必须实例化



### GC

GC roots:

* 本地变量表中引用的对象
* 方法区中静态变量引用的对象
* 方法去中常量引用的对象
* Native方法引用的对象

年轻代的GC是必须的，但老年代和永久代的GC并不是必须的，可以通过设置参数决定是否对类进行回收。



### JDK工具



### 正则表达式



### JAVA安全

#### java沙箱

java沙箱负责保护一些系统资源，保护的级别不同

- 内部资源，如本地内存
- 外部资源，如访问文件系统或者同一个局域网的其他机器
- 对于运行的组件applet，可以访问其web服务器
- 主机通过网络传输到磁盘的数据流

当前最新的安全机制实现，引入了domain概念，理解为将沙箱细分为对个具体的小沙箱。jvm把所有代码加载到不同的系统域和应用域。jvm中不同的受保护域，对应不一样的权限。存在于不同域中的类文件就具有当前域的全部权限。



沙箱的实现取决于三个方面：

- SecurityManager
- AccessController
- ClassLoader



#### SecurityManager

SecurityManager类是java API中的关键类，为其他java api提供相应的接口，是之可以检查某项操作能否执行。比如对文件访问的校验，SecurityManager最终是由AccessController实现的，如果权限检查不通过，则抛出AccessControlException，其继承于SecurityException，又向上继承于RuntimeException，所以这个异常是运行时异常。



对于不可信类，需要有一些方法避免它们绕过SecurityManger和Java API：

- checkCreateClassLoader
- checkExec
- checkLink
- checkExit
- checkPermission



#### AccessController

核心Java API由SecurityManager提供安全策略，但是大多数SecurityManager是基于AccessController。

AccessController类构造器是私有的，因此不能进行实例化。它向外部提供了一些静态方法，最关键的是checkPermission(Permission p)，该方法基于当前安装的Policy对象，判定当前保护域是否拥有指定权限。SecurityManger提供的额一系列checkxxxxx方法，最后基本通过这个checkPermission完成。



## 分散的知识

### switch支持的6种数据类型

只能是byte, short, char, int四种整形类型，enum和String类型。不能是boolean类型！

注意： 后面能能Integer只是因为JDK1.5以后的auto-unboxing



### Java整数缓存

Integer类为内部整数维护了IntgerCache, 范围是-128~127



### Thread.Sleep()和wait()的区别

- sleep方法执行时，线程主动让出cpu，后续指定时间以后cpu再回来执行。所以这里sleep只是让出了cpu，而并不会释放同步资源锁；wait方法指的是当前线程让自己暂时退让出同步资源锁，以便其他正在等待该资源的线程得到该资源从而运行，只有调用notify方法，调用了wait的线程才会解除wait状态，去竞争同步资源锁。
- 因此，sleep可以在任何地方使用，而wait只能在同步方法或者同步块中使用
- sleep是线程的方法，调用只会暂停该线程指定的时间，但是监控依然保持，不会释放对象锁，到时间自动回复。wait是对象的方法，调用会放弃对象锁，进入waitqueue，只有notify/notifyAll让唤醒指定的线程，进入锁池，再次获得对象锁才会进入运行态。



### Join方法让调用这个方法的线程执行完

A调用B的join，A会让出资源锁，B执行完了才会继续执行A



### JVM 吞吐量和暂停时间trade-off

吞吐量指的是应用程序线程用时占用程序用时的比例；暂停时间指的是一段时间内，应用程序线程让与GC线程执行而完全暂停，比如GC期间100ms的暂停时间意味着这个时间内没有应用程序线程是活动的。

这两者是矛盾的，如果要追求高吞吐量，就不能频繁调用GC，意味着GC启动的时候有很多事情要做，因为在此期间积累需要GC的对象比较多，所以应用程序暂停时间肯定长。



### JAVA接口成员变量和方法默认修饰符

成员变量的默认修饰符：public status final

- 也就是外面访问的时候不能修改这个变量的值，所以接口中定义成员变量的，一般都是常量

方法的默认修饰符：public abstract

- 公共抽象的，用来被实现这个接口的类去实现该方法

接口对修改关闭，对扩展开放，符合开闭原则。

**java8的接口方法可以被如下修饰**

- public, abstract, default, static, strictfp(只能修饰接口而不能修饰接口中的方法)



### 并发原子类

AtomicInteger类的常用

```java
// 获取当前值
public static void getCurrentValue(){}
// 设置value值
... setValue(){}
// 先获取旧值，再设置新值
... getAndSet(){}
// 先获取旧值，再进行自增
... getAndIncrement(){}
// 同理
... getAndDecrement(){}
// 先获取旧值，再加10
... getAndAdd(){}
// 先加1，再获取新值
... incrementAmdGet(){}
// 先减1，再获取新值
... decrementAndGet(){}
// 先增加，再获取新值
... addAndGet(){}
```

concurrent包还有其他:

- AtomicBoolean
- AtomicInteger
- AtomicIntegerArray
- AtomicIntegerFieldUpdater
- AtomicLong
- AtomicLongArray
- AtomicIongFieldUpdater
- AtomicReference
- AtomicReferenceArray
- AtomicReferenceFieldUpdater

....

### Java CountDownLatch方法并发编程



### Java 类执行顺序

- 执行static代码块
- 执行构造代码块
- 执行构造函数

静态变量、初始化块 > 变量、初始化块 > 构造器

涉及到继承的时候：

- 执行父类的静态代码块，初始化父类静态成员变量
- 执行子类静态代码块，初始化子类静态成员变量
- 执行父类的构造代码块，执行父类的构造函数，并初始化父类普通成员变量
- 执行子类的构造代码块，执行子类构造函数，初始化子类普通成员变量

总结：涉及到继承，静态的优先，其他的先父类执行完了再子类



### 线程安全的数据结构

- V （Vector）
- S   (Stack)
- H   (HashTable)
- E   (Enumeration)

