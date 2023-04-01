---
title: 【Java】Java多线程
date: 2023-3-29
tags: [Java,多线程]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/2023_03_27_2.webp
mathjax: true
---
# Java多线程
## 1. 创建线程
### 1.1 创建线程类
Java使用`java.lang.Thread`类代表**线程**，所有的线程对象都必须是Thread类或其子类的实例。每个线程的作用是完成一定的任务，实际上就是执行一段程序流即一段顺序执行的代码。Java使用线程执行体来代表这段程序流。

创建新执行线程有两种方法。一种方法是将类声明为 `Thread` 的子类。该子类应重写 `Thread` 类的  `run` 方法。接下来可以分配并启动该子类的实例。创建线程的另一种方法是声明实现 `Runnable` 接口的类。该类然后实现 `run`  方法。然后可以分配该类的实例，在创建 `Thread` 时作为一个参数来传递并启动。

Java中通过继承Thread类来**创建**并**启动多线程**的步骤如下：

1. 定义Thread类的子类，并重写该类的run()方法，该run()方法的方法体就代表了线程需要完成的任务,因此把run()方法称为线程执行体。
2. 创建Thread子类的实例，即创建了线程对象
3. 调用线程对象的start()方法来启动该线程

代码如下：

测试类：
```java
public class Demo01 {
	public static void main(String[] args) {
		//创建自定义线程对象
		MyThread mt = new MyThread("新的线程！");
		//开启新线程
		mt.start();
		//在主方法中执行for循环
		for (int i = 0; i < 10; i++) {
			System.out.println("main线程！"+i);
		}
	}
}
```

自定义线程类：

```java
public class MyThread extends Thread {
	//定义指定线程名称的构造方法
	public MyThread(String name) {
		//调用父类的String参数的构造方法，指定线程的名称
		super(name);
	}
	/**
	 * 重写run方法，完成该线程执行的逻辑
	 */
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			System.out.println(getName()+"：正在执行！"+i);
		}
	}
}
```

Notice：

+ start()使该线程开始执行；Java 虚拟机调用该线程的 run 方法。 结果是两个线程并发地运行；当前线程（从调用返回给 start 方法）和另一个线程（执行其 run 方法）。 多次启动一个线程是非法的。特别是当线程已经结束执行后，不能再重新启动。
+ Java程序属于抢占式调度，哪个线程优先级高就优先执行哪个线程，优先级一样随机选择

### 1.2  多线程内存图解

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202303292248208.webp)

### 1.3 声明实现Runnable接口的实现类

实现类

```java
public class RunnableImpl implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            System.out.println(Thread.currentThread().getName() + ":" + i);
        }
    }
}
```

测试类：

```java
public class DemoRunnable {
    public static void main(String[] args) {
        RunnableImpl run = new RunnableImpl();
        Thread t = new Thread(run);
        t.start();
        for (int i = 0; i < 20; i++) {
            System.out.println(Thread.currentThread().getName() + ":" + i);
        }
    }
}
```

**Thread和Runnable的区别**

实现Runnable接口创建多线程程序的好处:

+ 避免了单继承的局限性，一个类只能继承一个类，类如果继承了Thread类就不能继承其他类，实现了Runnable接口还可以继承其他的类。
+ 增强了程序的扩展性，降低程序的耦合性，实现Runnable接口的方式，把设置线程任务和开启新线程进行了分离。实现类中，重写了run方法，用来设置线程任务。创建Thread类对象，调用start方法，用来开启新线程。给Thread传递不同的实现类可以实现不同的任务。

### 1.4 匿名内部类方式实现线程的创建

使用匿名内部类可以简化代码，将子类继承父类，重写父类的方法，创建子类对象合成为一步完成。

```java
格式: new 父类/接口(){
    重写父类/接口中的方法
};
```

样例代码：

```java
public class DemoInnerClassThread {
    public static void main(String[] args) {
        new Thread() {
            @Override
            public void run() {
                int i = 0;
                while (i < 20) {
                    System.out.println(Thread.currentThread().getName() + ":" + i);
                    i++;
                }
            }
        }.start();
        //线程的接口Runnable
        //Runnable r = new Runnable();//多态
        //使用接口接受一个实现类
        Runnable r = new Runnable() {
            @Override
            public void run() {
                int i = 0;
                while (i < 20) {
                    System.out.println(Thread.currentThread().getName() + ":" + i);
                    i++;
                }
            }
        };
        new Thread(r).start();
    }
}
```

## 2. 线程同步

### 2.1 线程安全问题

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202303292248521.webp)

存在线程安全问题的代码样例:

```java
/*
 * 实现卖票案例
 * */
public class RunnableImpl implements Runnable {
    //定义一个多线程共享的票源
    private int ticket = 100;

    //设置线程任务:卖票
    @Override
    public void run() {
        //让卖票操作重复执行
        while (true) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //先判断票还有没有
            if (ticket > 0) {
                //票存在,卖票
                System.out.println(Thread.currentThread().getName() + "->正在卖第" + ticket + "张票");
                ticket--;
            }
        }
    }
}

```

测试类:

```java
public class Demo01Ticket {
    public static void main(String[] args) {
        //创建Runnable接口的实现类对象
        RunnableImpl run = new RunnableImpl();
        Thread t0 = new Thread(run);
        Thread t1 = new Thread(run);
        Thread t2 = new Thread(run);
        t0.start();
        t1.start();
        t2.start();
    }
}
```
可以看到出现了重复
```
Thread-0->正在卖第100张票
Thread-2->正在卖第100张票
Thread-1->正在卖第100张票
Thread-2->正在卖第97张票
Thread-0->正在卖第97张票
Thread-1->正在卖第97张票
Thread-1->正在卖第94张票
Thread-2->正在卖第94张票
......
```

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202303292248298.webp)

### 2.2 线程同步问题

Java提供了线程同步机制(synchronized)来解决线程安全问题

有3种方式完成同步操作:

+ 同步代码块
+ 同步方法
+ 锁机制

#### 2.2.1 同步代码块

**同步代码块**: `synchronized`关键字可以用于方法中的某个区块中，表示只对这个区块的资源进行互斥访问.

```java
格式:
synchronized(同步锁){
    需要同步操作的代码块
}
```

**同步锁**:

对象的同步锁只是一个概念，可以想象为在对象上标记了一个锁

1. 锁对象可以是任意类型。

2. 多个线程对象只能使用同一把锁。
3. 作用: 把同步代码块锁住，只让一个线程在同步代码块中执行。

```java
public class RunnableImpl implements Runnable {
    //定义一个多线程共享的票源
    private int ticket = 100;
    Object lock = new Object();

    //设置线程任务:卖票
    @Override
    public void run() {
        //让卖票操作重复执行
        while (true) {
            synchronized (lock) {
                //先判断票还有没有
                if (ticket > 0) {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    //票存在,卖票
                    System.out.println(Thread.currentThread().getName() + "->正在卖第" + ticket + "张票");
                    ticket--;
                }
            }
        }
    }
}
```

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202303292248264.webp)

#### 2.2.2 同步方法

**同步方法**:使用synchronized修饰的方法，就叫做同步方法，保证A线程执行该方法的时候，其他线程只能在外面等着。

```java
格式:
public synchronized void method(){
    可能会产生线程安全问题的代码
}
```

同步锁是谁：

+ 对于非static方法，同步锁就是this

+ 对于static方法，同步锁就是当前方法所在类的字节码对象(类名.class)

```java
/*
 * 实现卖票案例,使用同步方法解决线程同步问题
 * */
public class RunnableImpl implements Runnable {
    //定义一个多线程共享的票源
    private int ticket = 100;
    //设置线程任务:卖票
    @Override
    public void run() {
        //让卖票操作重复执行
        while (true) {
            payTicket();
        }
    }
    //定义一个同步方法
    public synchronized void payTicket() {
        //先判断票还有没有
        if (ticket > 0) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //票存在,卖票
            System.out.println(Thread.currentThread().getName() + "->正在卖第" + ticket + "张票");
            ticket--;
        }
    }
}
```

#### 2.2.3 Lock锁

java.util.concurrent.locks 

`Lock` 实现提供了比使用 `synchronized`  方法和语句可获得的更广泛的锁定操作。同步代码块和同步方法具有的功能Lock锁都具备，除此之外更加强大，更加体现面向对象。

Lock锁也称同步锁，加锁和释放锁方法化了，如下:

+ `public void lock()`:加同步锁
+ `public void unlock()`:释放同步锁

```java
//java.util.concurrent.locks.lock//接口
//java.util.concurrent.locks.ReentrantLock//实现类
使用步骤:
1. 在成员位置创建一个ReentrantLock对象
2. 在可能出现线程安全问题的代码前调用Lock接口中的Lock方法获取锁
2. 在可能出现线程安全的代码后调用Lock接口中的unlock方法释放锁
```

使用如下:

```java
/*
 * 实现卖票案例,使用Lock锁解决线程同步问题
 * java.util.concurrent.locks.lock接口
 * java.util.concurrent.locks.ReentrantLock实现类
 * */
public class RunnableImpl implements Runnable {
    //定义一个多线程共享的票源
    private int ticket = 100;
    Lock lock = new ReentrantLock();

    //设置线程任务:卖票
    @Override
    public void run() {
        //让卖票操作重复执行
        while (true) {
            lock.lock();
            //先判断票还有没有
            if (ticket > 0) {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //票存在,卖票
                System.out.println(Thread.currentThread().getName() + "->正在卖第" + ticket + "张票");
                ticket--;
            }
            lock.unlock();
        }
    }
}
```
## 3. 线程状态

当线程被创建并启动以后，它既不是一启动就进入了执行状态，也不是一直处于执行状态。在线程的生命周期中， 有几种状态呢？在API中` java.lang.Thread.State` 这个枚举中给出了六种线程状态： 这里先列出各个线程状态发生的条件，下面将会对每种状态进行详细解析:

| 线程状态                | 导致状态发生的条件                                           |
| ----------------------- | ------------------------------------------------------------ |
| NEW(新建)               | 线程刚被创建，但是并未启动。还未调用start方法                |
| Runnable(可运行)        | 线程可以在jvm中运行的状态，可能正在运行自己的代码，也可能没有，看OS和CPU |
| Blocked(锁阻塞)         | 当一个线程试图获取一个对象锁，而该对象锁被其他线程持有，则该线程进入Blocked状态，当该线程持有锁时，将变成Runnable状态 |
| Waiting(无限等待)       | 一个线程在等待另一个线程执行(唤醒)操作时，该线程进入waiting状态。进入这个状态后是不能自动唤醒的，必须等待另一个线程调用notify或者notifyAll方法才能够唤醒。 |
| Timed Waiting(计时等待) | 同waiting状态，有几个方法有超时参数，调用他们将进入Timed Waiting状态。这一状态将一直保持到超时期满或者接收到唤醒通知。带有超时参数的常用方法有Thread.sleep、Obejct.wait。 |
| Terminated(被终止)      | 因为run方法正常退出而死亡,或者因为没有捕获的异常终止了run方法而死亡。 |

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%8A%B6%E6%80%81%E5%9B%BE.webp)

### 3.1 Timed Waiting(计时等待状态)

Timed Waiting在API中的描述为：一个正在限时等待另一个线程执行一个（唤醒）动作的线程处于这一状态。在我们写卖票的案例中，为了减少线程执行太快，现象不明显等问题，我们在run方法中添加了sleep语句，这样就 强制当前正在执行的线程休眠（暂停执行）以“减慢线程”。其实当我们调用了sleep方法之后，当前执行的线程就进入到“休眠状态”，其实就是所谓的Timed Waiting(计时等待)。

<img src="https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/%E8%AE%A1%E6%97%B6%E7%AD%89%E5%BE%85.webp"  />

### 3.2 BLOCKED(锁阻塞)
Blocked状态在API中的介绍为：一个正在阻塞等待一个监视器锁（锁对象）的线程处于这一状态。
我们已经学完同步机制，那么这个状态是非常好理解的了。比如，线程A与线程B代码中使用同一锁，如果线程A获取到锁，线程A进入到Runnable状态，那么线程B就进入到Blocked锁阻塞状态。

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/%E9%94%81%E9%98%BB%E5%A1%9E.webp)


## 4. 等待与唤醒机制
### 4.1 线程间通信

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/%E7%BA%BF%E7%A8%8B%E9%97%B4%E9%80%9A%E4%BF%A1.webp)

**为什么要处理线程间通信:**

多个线程并行执行时，在默认情况下CPU是随即切换线程的，当我们需要多个线程来共同完成一件任务，并且我们希望他们有规律的执行，那么多线程之间需要一些协调通信，以此来帮助我们达到多线程共同操作一份数据。

**如何保证线程间通信有效利用资源：**

多个线程在处理同一个资源，并且任务不同时，需要线程通信来帮助解决线程之间对同一个变量的使用或操作。 就是多个线程在操作同一份数据时， 避免对同一共享变量的争夺。也就是我们需要通过一定的手段使各个线程能有效的利用资源。而这种手段即—— **等待唤醒机制。**


### 4.2  等待与唤醒机制

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/01_%E7%AD%89%E5%BE%85%E4%B8%8E%E5%94%A4%E9%86%92%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90_1.webp)

**什么是等待唤醒机制**

这是多个线程间的一种**协作**机制。谈到线程我们经常想到的是线程间的**竞争（race）**，比如去争夺锁，但这并不是故事的全部，线程间也会有协作机制。就好比在公司里你和你的同事们，你们可能存在在晋升时的竞争，但更多时候你们更多是一起合作以完成某些任务。

就是在一个线程进行了规定操作后，就进入等待状态（**wait()**）， 等待其他线程执行完他们的指定代码过后 再将其唤醒（**notify()**）;在有多个线程进行等待时， 如果需要，可以使用 notifyAll()来唤醒所有的等待线程。

wait/notify 就是线程间的一种协作机制。

**等待唤醒中的方法**

等待唤醒机制就是用于解决线程间通信的问题的，使用到的3个方法的含义如下：

1. wait：线程不再活动，不再参与调度，进入 wait set 中，因此不会浪费 CPU 资源，也不会去竞争锁了，这时的线程状态即是 WAITING。它还要等着别的线程执行一个**特别的动作**，也即是“**通知（notify）**”在这个对象上等待的线程从wait set 中释放出来，重新进入到调度队列（ready queue）中
2. notify：则选取所通知对象的 wait set 中的一个线程释放；例如，餐馆有空位置后，等候就餐最久的顾客最先入座。
3. notifyAll：则释放所通知对象的 wait set 上的全部线程。

>注意：
>
>哪怕只通知了一个等待的线程，被通知线程也不能立即恢复执行，因为它当初中断的地方是在同步块内，而此刻它已经不持有锁，所以她需要再次尝试去获取锁（很可能面临其它线程的竞争），成功后才能在当初调用 wait 方法之后的地方恢复执行。
>
>总结如下：
>
>- 如果能获取锁，线程就从 WAITING 状态变成 RUNNABLE 状态；
>- 否则，从 wait set 出来，又进入 entry set，线程就从 WAITING 状态又变成 BLOCKED 状态

**调用wait和notify方法需要注意的细节**

1. wait方法与notify方法必须要由同一个锁对象调用。因为：对应的锁对象可以通过notify唤醒使用同一个锁对象调用的wait方法后的线程。
2. wait方法与notify方法是属于Object类的方法的。因为：锁对象可以是任意对象，而任意对象的所属类都是继承了Object类的。
3. wait方法与notify方法必须要在同步代码块或者是同步函数中使用。因为：必须要通过锁对象调用这2个方法。

### 4.3 生产者与消费者问题

等待唤醒机制其实就是经典的生产者消费者问题。

以生产包子消费包子为例来说明等待唤醒机制如何有效利用资源：

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/01_%E7%AD%89%E5%BE%85%E4%B8%8E%E5%94%A4%E9%86%92%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90_2.webp)
包子类

```java
public class BaoZi {
    //皮
    String skin;
    //馅
    String stuff;
    //包子的状态:true 有 false无
    boolean flag = false;
}
```
包子铺类

```java
/*
 * 注意: 包子铺线程和包子线程的关系---(互斥)
 * 必须同时同步，保证两个线程只有一个在执行
 * 锁对象必须保持唯一，可以使用包子对象作为锁对象
 * 包子铺类和吃货类需要把包子对象作为参数传进来
 * */
public class BaoZiStore extends Thread {
    private BaoZi bz;

    public BaoZiStore(BaoZi bz) {
        this.bz = bz;
    }

    @Override
    public void run() {
        //定义一个变量用来记录包子的种类
        int cnt = 0;
        while (true) {
            synchronized (bz) {
                //对包子的状态进行判断
                if (bz.flag) {
                    //包子铺调用wait方法进入等待状态
                    try {
                        bz.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //被唤醒后执行
                //交替生产两种包子
                if (cnt % 2 == 0) {
                    //生产薄皮三鲜馅包子
                    bz.skin = "薄皮";
                    bz.stuff = "三鲜";
                } else {
                    bz.skin = "冰皮";
                    bz.stuff = "牛肉";
                }
                cnt++;
                System.out.println("包子铺正在生产:" + bz.skin + bz.stuff);
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //包子铺生产好了包子
                //修改包子的状态为true
                bz.flag = true;
                //唤醒等待的吃货线程
                bz.notify();
                System.out.println("包子铺已经生产好了:" + bz.skin + bz.stuff + "吃货可以开始吃了");
            }
        }
    }
}
```

消费者类

```java
public class Consumer extends Thread {
    private BaoZi bz;

    public Consumer(BaoZi bz) {
        this.bz = bz;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (bz) {
                //对包子状态进行判断
                if (!bz.flag) {
                    //吃货调用wait方法进入等待状态
                    try {
                        bz.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //被唤醒后执行的代码
                System.out.println("吃货正在吃:" + bz.skin + bz.stuff);
                bz.flag = false;
                bz.notify();
                System.out.println("吃货已经把:" + bz.skin + bz.stuff + "吃完了，包子铺开始生产包子了");
                System.out.println("__________________________________________________");
            }
        }
    }
}
```

测试类

```java
public class test {
    public static void main(String[] args) {
        BaoZi bz = new BaoZi();
        new BaoZiStore(bz).start();
        new Consumer(bz).start();
    }
}
```



## 5. 线程池

### 5.1 线程池概念

**线程池：**其实就是一个容纳多个线程的容器，其中的线程可以反复使用，省去了频繁创建线程对象的操作，无需反复创建线程而消耗过多资源。

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/02_%E7%BA%BF%E7%A8%8B%E6%B1%A0.webp)

合理利用线程池能够带来三个好处：

1. 降低资源消耗。减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。
2. 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
3. 提高线程的可管理性。可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)。

### 5.2 线程池的使用

Java里面线程池的顶级接口是`java.util.concurrent.Executor`，但是严格意义上讲`Executor`并不是一个线程池，而只是一个执行线程的工具。真正的线程池接口是`java.util.concurrent.ExecutorService`。

要配置一个线程池是比较复杂的，尤其是对于线程池的原理不是很清楚的情况下，很有可能配置的线程池不是较优的，因此在`java.util.concurrent.Executors`线程工厂类里面提供了一些静态工厂，生成一些常用的线程池。官方建议使用Executors工程类来创建线程池对象。

Executors类中有个创建线程池的方法如下：

* `public static ExecutorService newFixedThreadPool(int nThreads)`：返回线程池对象。(创建的是有界线程池,也就是池中的线程个数可以指定最大数量)

获取到了一个线程池ExecutorService 对象，那么怎么使用呢，在这里定义了一个使用线程池对象的方法如下：

* `public Future<?> submit(Runnable task)`:获取线程池中的某一个线程对象，并执行

  > Future接口：用来记录线程任务执行完毕后产生的结果。线程池创建与使用。

```java
/**
 * 线程池jdk1.5之后提供
 * java.util.concurrent.Executors:线程池的工厂类，用来生成线程池
 * Executor类中的静态方法
 *      static ExecutorService newFixedThreadPool(int nThreads)创建一个可重用固定线程数的线程池
 * 参数:
 *      int nThreads:创建线程池中的线程数量
 * 返回值:
 *      ExecutorService接口，返回的是ExecutorService接口的实现对象，我们可以使用ExecutorService接口接收(面向接口编程)
 * java.util.concurrent.ExecutorService:线程池接口
 *      用来从线程池中获取线程，调用start方法，执行线程任务
 *          submit(Runnable task)提交一个Runnable任务用于执行
 *      关闭/销毁线程池的方法
 *          void shutdown()按顺序完成当前任务,不在接收新任务
 * 线程池的使用步骤:
 *     1.使用线程池的工厂类Executors里边提供的静态方法newFixedThreadPool生产一个指定线程数量的线程池
 *     2.创建一个类，实现Runnable接口，重写run方法
 *     3.调用ExecutorService中的方法submit,传递线程任务(实现类),开启线程，执行run方法
 *     4.(可选)调用ExecutorService中的方法shutdown销毁线程池
 */
```

使用线程池中线程对象的步骤：

1. 创建线程池对象。
2. 创建Runnable接口子类对象。(task)
3. 提交Runnable接口子类对象。(take task)
4. 关闭线程池(一般不做)。

Runnable实现类代码：

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("我要一个教练");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("教练来了： " + Thread.currentThread().getName());
        System.out.println("教我游泳,交完后，教练回到了游泳池");
    }
}
```

线程池测试类：

```java
public class ThreadPoolDemo {
    public static void main(String[] args) {
        // 创建线程池对象
        ExecutorService service = Executors.newFixedThreadPool(2);//包含2个线程对象
        // 创建Runnable实例对象
        MyRunnable r = new MyRunnable();

        //自己创建线程对象的方式
        // Thread t = new Thread(r);
        // t.start(); ---> 调用MyRunnable中的run()

        // 从线程池中获取线程对象,然后调用MyRunnable中的run()
        service.submit(r);
        // 再获取个线程对象，调用MyRunnable中的run()
        service.submit(r);
        service.submit(r);
        // 注意：submit方法调用结束后，程序并不终止，是因为线程池控制了线程的关闭。
        // 将使用完的线程又归还到了线程池中
        // 关闭线程池
        //service.shutdown();
    }
}
```



