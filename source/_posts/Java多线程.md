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

