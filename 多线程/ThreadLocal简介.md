## ThreadLocal简介

<font size=2>摘自《架构探险-从0开始写Java Web框架》 p158</font>

### 什么是ThreadLocal

ThreadLocal直译为“线程本地”或“本地线程”，如果这么认为，那就错了！其实它就是一个容器，用于存放线程的局部变量，应该叫ThreadLocalVariable(线程局部变量)才对，很不理解为什么当初Sun工程师这样命名。

早在JDK1.2的时代，java.lang.ThreadLocal就诞生了，它是为了解决多线程并发问题而设计的，只不过设计的有些难用而已，所以至今没有得倒广泛使用。

一个序列号生成器的程序可能同时会有多个线程并发访问它，要保证每个线程得倒的序列号都是自增的，而不能相互干扰。

先定义一个接口：

```java
public interface Sequence {
    int getNumber();
}
```



每次调用getNumber方法可获取一个序列号，下次再调用时，序列号会自增。

再做一个线程类：

```java
public class ClientThread extends Thread {
    private Sequence sequence;

    public ClientThread(Sequence sequence) {
        this.sequence = sequence;
    }

    @Override
    public void run() {
        for (int i = 0; i < 3; i++) {
            System.out.println(
                    Thread.currentThread().getName() + " =>" + sequence.getNumber());
        }
    }
}
```

我们不用ThreadLocal，先做一个实现类：

```java
public class SequenceA implements Sequence {
    private static int number = 0;
    
    @Override
    public int getNumber() {
        number = number + 1;
        return number;
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceA();
        
        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);
        
        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

序列号初始值是0，在main方法中模拟了三个线程，运行后结果如下：

```java
Thread-0 =>1
Thread-0 =>2
Thread-0 =>3
Thread-1 =>4
Thread-2 =>5
Thread-1 =>6
Thread-2 =>7
Thread-1 =>8
Thread-2 =>9
```

由于线程启动顺序是随机的，所以并不是0、1、2这样的顺序，这个好理解。为什么当Thread-0输出了1、2、3后，而Thread-1、Thread-2却输出了4、5、6呢？不应该从0开始输出吗？

仔细分析才发现，线程之间共享static变量无法保证对不同线程而言是安全的，也就是说，此时无法保证“线程安全”。

那么如何才能做到“线程安全”呢？对应于这个案例，就是说不同的线程可拥有自己的static变量，如何实现呢？下面看看另外一个示例：

```java
public class SequenceB implements Sequence {
    private static ThreadLocal<Integer> numberContainer = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    @Override
    public int getNumber() {
        numberContainer.set(numberContainer.get() + 1);
        return numberContainer.get();
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceB();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

通过ThreadLocal封装了一个Integer类型的numberContainer静态成员变量，并且初始值是0。再看getNumber方法，首先从numberContainer中get出当前的值，加1，随后set到numberContainer中，最后在numberContainer中get出当前的值并返回。

是不是很绕？但是很强大！我们不妨把ThreadLocal看作是一个容器，这样理解起来就简单了，所以，这里故意用Container这个单词作为后缀来命名ThreadLocal变量。

运行结果如下：

```java
Thread-0 =>1
Thread-0 =>2
Thread-0 =>3
Thread-2 =>1
Thread-2 =>2
Thread-2 =>3
Thread-1 =>1
Thread-1 =>2
Thread-1 =>3
```

每个线程相互独立了，同样是static变量，对于不同的线程而言，它没有被共享，而是每个线程各一份，这样也就保证了线程安全。也就是说，ThreadLocal为每一个线程提供了一个独立的副本。

搞清楚ThreadLocal的原理之后，有必要总结一下ThreadLocal的API，其实很简单。

- public void set(T value): 将值放入线程局部变量中；
- public T get(): 从线程局部变量中获取值；
- public void remove()：从线程局部变量中移除值（有助于JVM垃圾回收）
- protected T initialValue():返回线程局部变量中的初始值(默认为null)

为什么initalValue方法是protected的呢？就是为了提醒程序员，这个方法是要程序员来实现的，要给这个局部变量设置一个初始值。



#### 自己实现ThreadLocal

熟悉了原理与这些API之后，其实想想ThreadLocal里面不就是封装了一个Map吗？自己都可以写一个ThreadLocal了：

```java
public class MyThreadLocal<T> {
    private Map<Thread, T> container = Collections.synchronizedMap(new HashMap<>());

    public void set(T value) {
        container.put(Thread.currentThread(), value);
    }

    public T get() {
        Thread thread = Thread.currentThread();
        T value = container.get(thread);
        if (value == null && !container.containsKey(thread)) {
            value = initialValue();
            container.put(thread, value);
        }
        return value;
    }

    public void remove() {
        container.remove(Thread.currentThread());
    }

    protected T initialValue() {
        return null;
    }
}
```

以上完全“山寨”了一个ThreadLocal，其中定义了一个同步Map，下面用MyThreadLocal再来实现一次：

```java
public class SequenceC implements Sequence {

    private static MyThreadLocal<Integer> numberContainer = new MyThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };

    @Override
    public int getNumber() {
        numberContainer.set(numberContainer.get() + 1);
        return numberContainer.get();
    }

    public static void main(String[] args) {
        Sequence sequence = new SequenceC();

        ClientThread thread1 = new ClientThread(sequence);
        ClientThread thread2 = new ClientThread(sequence);
        ClientThread thread3 = new ClientThread(sequence);

        thread1.start();
        thread2.start();
        thread3.start();
    }
}
```

运行结果：

```java
Thread-0 =>1
Thread-0 =>2
Thread-0 =>3
Thread-2 =>1
Thread-2 =>2
Thread-2 =>3
Thread-1 =>1
Thread-1 =>2
Thread-1 =>3
```

以上代码其实就是将ThreadLocal替换成了MyThreadLocal，仅此而已，运行效果和之前一样，也是正确的。