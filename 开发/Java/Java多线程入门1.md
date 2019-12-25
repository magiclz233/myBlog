### Java多线程入门1

#### 进程与线程

在讲多线程之前，我觉得有必要先说一下进程与线程之间的关系与差异。

 1、进程是资源分配的最小单位，线程是程序执行的最小单位（资源调度的最小单位）
 2、进程有自己的独立地址空间，每启动一个进程，系统就会为它分配地址空间，建立数据表来维护代码段、堆栈段和数据段，这种操作非常昂贵。
 而线程是共享进程中的数据的，使用相同的地址空间，因此CPU切换一个线程的花费远比进程要小很多，同时创建一个线程的开销也比进程要小很多。
 3、线程之间的通信更方便，同一进程下的线程共享全局变量、静态变量等数据，而进程之间的通信需要以通信的方式（IPC)进行。不过如何处理好同步与互斥是编写多线程程序的难点。
 4、但是多进程程序更健壮，多线程程序只要有一个线程死掉，整个进程也死掉了，而一个进程死掉并不会对另外一个进程造成影响，因为进程有自己独立的地址空间。

通俗点来讲，进程就像是任务管理器中的qq，chrome，网易云音乐这种一个个应用，而线程就像是在这个进程中间的一次任务，比如你点击切换音乐，聊天发送信息等。

#### 多线程的实现

在Java中多线程的实现有三种形式，这里只说前两种，继承Thread类和实现Runnable接口。

##### 1 继承Thread类

```Java
//继承Thread实现多线程
class Thread1 extends Thread{  
    private String name;  
    public Thread1(String name) {  
       this.name=name;  
    }  
    public void run() {  
        for (int i = 0; i < 5; i++) {  
            System.out.println(name + "运行  :  " + i);  
            try {  
                sleep((int) Math.random() * 10);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        }  
         
    }  
}  
public class Main {  
  
    public static void main(String[] args) {  
        Thread1 mTh1=new Thread1("A");  
        Thread1 mTh2=new Thread1("B");  
        mTh1.start();  
        mTh2.start();  
  
    }  
}  
```

上面这个两个类，`Thread1`类继承了Thread父类，并重写了里面的`run`方法。实现了多线程里面的方法，并在main函数中进行实例化了两个`mTh1`,`mTh2`两个线程。

启动main函数：

```Java
输出：
A运行 : 0
B运行 : 0
A运行 : 1
A运行 : 2
A运行 : 3
A运行 : 4
B运行 : 1
B运行 : 2
B运行 : 3
B运行 : 4
```

再运行一下：

```
A运行 : 0
B运行 : 0
B运行 : 1
B运行 : 2
B运行 : 3
B运行 : 4
A运行 : 1
A运行 : 2
A运行 : 3
A运行 : 4
```

可以看到两次运行的结果是不太一样的。

>**说明**
>
>程序在启动main函数时，Java虚拟机就已经启动了一个主线程来运行main函数，在调用到`mTh1`，`mTh2`的start方法时，就相当于有三个线程在同时工作了，这就是多线程的模式，进入了`mTh1`子线程，这个线程中的操作，在这个线程中有`sleep()`方法，Thread.sleep()方法调用目的是不让当前线程独自霸占该进程所获取的CPU资源，以留出一定时间给其他线程执行的机会。

实际上所有的线程执行顺序都是不确定的，CPU资源的获取完全是看两个线程之间谁先抢占上谁就先运行，当`mTh1`抢占上线程后，运行`run`方法中的代码，到`sleep()`方法进入休眠状态，也就是阻塞状态，然后`CPU`资源会被释放，`A`，`B`再次进行抢占`CPU`资源操作，抢占上的继续运行。在运行的结果中你也可以看到这个现象。

注意:

一个实例的start()方法不能重复调用，否则会出现`java.lang.IllegalThreadStateException`异常。

##### 2 实现`java.lang.Runnable`接口

采用Runnable也是非常常见的一种，我们只需要重写run方法即可。下面也来看个实例。

```java
class Thread2 implements Runnable{  
    private String name;  
  
    public Thread2(String name) {  
        this.name=name;  
    }  
  
    @Override  
    public void run() {  
          for (int i = 0; i < 5; i++) {  
                System.out.println(name + "运行  :  " + i);  
                try {  
                    Thread.sleep((int) Math.random() * 10);  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                }  
            }  
          
    }  
      
}  
public class Main {  
  
    public static void main(String[] args) {  
        new Thread(new Thread2("C")).start();  
        new Thread(new Thread2("D")).start();  
    }  
  
}  
```

整体和继承Thread差别不大，因为在Thread类中也是继承的Runnable接口。

输出运行：

```
C运行 : 0
D运行 : 0
D运行 : 1
C运行 : 1
D运行 : 2
C运行 : 2
D运行 : 3
C运行 : 3
D运行 : 4
C运行 : 4
```

**说明：**

>`Thread2`类通过实现`Runnable`接口，使得该类有了多线程类的特征。`run()`方法是多线程程序的一个约定。所有的多线程代码都在`run`方法里面。`Thread`类实际上也是实现了`Runnable`接口的类。
>
>在启动的多线程的时候，需要先通过`Thread`类的构造方法`Thread(Runnable target) `构造出对象，然后调用Thread对象的start()方法来运行多线程代码。
>
>实际上所有的多线程代码都是通过运行`Thread`的`start()`方法来运行的。因此，不管是扩展Thread类还是实现`Runnable`接口来实现多线程，最终还是通过`Thread`的对象的`API`来控制线程的，熟悉`Thread`类的`API`是进行多线程编程的基础。

#### Thread类和Runnable接口的区别

如果一个类继承`Thread`，则不适合资源共享。但是如果实现了`Runable`接口的话，则很容易的实现资源共享。

**总结：**

**实现`Runnable`接口比继承`Thread`类所具有的优势：**

**1）：适合多个相同的程序代码的线程去处理同一个资源**

**2）：可以避免`java`中的单继承的限制**

**3）：增加程序的健壮性，代码可以被多个线程共享，代码和数据独立**

**4）：线程池只能放入实现`Runable`或`callable`类线程，不能直接放入继承`Thread`的类**

**提醒一下大家：main方法其实也是一个线程。在`java`中所以的线程都是同时启动的，至于什么时候，哪个先执行，完全看谁先得到CPU的资源。**

**在`java`中，每次程序运行至少启动2个线程。一个是`main`线程，一个是垃圾收集线程。因为每当使用`java`命令执行一个类的时候，实际上都会启动一个`JVM`，每一个`JVM`实习在就是在操作系统中启动了一个进程。**

#### 线程的状态

下面先放一张线程的展示图

![img](https://upload-images.jianshu.io/upload_images/6490792-ef23c852823a4401.jpeg?imageMogr2/auto-orient/)

1：新建状态(New)：new Thread()，新创建了一个线程；

2：就绪状态(Runnable)：新建完成后，主线程(main()方法)调用了该线程的start()方法，CPU目前在执行其他任务或者线程，这个创建好的线程就会进入就绪状态，等待CPU资源运行程序，在运行之前的这段时间处于就绪状态；

3：运行状态(Running)：字面意思，线程调用了start()方法之后并且抢占到了CPU资源，运行run方法中的程序代码；

4：阻塞状态(Blocked)：阻塞状态时线程在运行过程中因为某些操作暂停运行，放弃CPU使用权，进入就绪状态和其他线程一同进行下次CPU资源的抢占。

当发生如下情况时，线程将会进入阻塞状态

　　① 线程调用sleep()方法主动放弃所占用的处理器资源

　　② 线程调用了一个阻塞式IO方法，在该方法返回之前，该线程被阻塞

　　③ 线程试图获得一个同步监视器，但该同步监视器正被其他线程所持有。关于同步监视器的知识、后面将有深入的介绍

　　④ 线程在等待某个通知（notify）

　　⑤ 程序调用了线程的suspend()方法将该线程挂起。**但这个方法容易导致死锁，所以应该尽量避免使用该方法**

　　当前正在执行的线程被阻塞之后，其他线程就可以获得执行的机会。被阻塞的线程会在合适的时候重新进入就绪状态，注意是就绪状态而不是运行状态。也就是说，被阻塞线程的阻塞解除后，必须重新等待线程调度器再次调度它。

**解除阻塞**

　　针对上面几种情况，当发生如下特定的情况时可以解除上面的阻塞，让该线程重新进入就绪状态：

　　① 调用sleep()方法的线程经过了指定时间。

　　② 线程调用的阻塞式IO方法已经返回。

　　③ 线程成功地获得了试图取得的同步监视器。

　　④ 线程正在等待某个通知时，其他线程发出了个通知。

　　⑤ 处于挂起状态的线程被调甩了`resdme()`恢复方法(会导致死锁，尽量避免使用)。

5：死亡状态(Dead)：线程程序执行完成或者因为发生异常跳出了run()方法，线程生命周期结束。

####  线程的调度

 1：调整线程优先级：`Java`线程有优先级，优先级高的线程会获得较多的运行机会。
` Java`线程的优先级用整数表示，取值范围是1~10，`Thread`类有以下三个静态常量：
` static int MAX_PRIORITY`
 线程可以具有的最高优先级，取值为10。
 `static int MIN_PRIORITY`
 线程可以具有的最低优先级，取值为1。
 `static int NORM_PRIORITY`
 分配给线程的默认优先级，取值为5。
 `Thread`类的`setPriority()`和`getPriority()`方法分别用来设置和获取线程的优先级。
 每个线程都有默认的优先级。主线程的默认优先级为`Thread.NORM_PRIORITY`。
 线程的优先级有继承关系，比如A线程中创建了B线程，那么B将和A具有相同的优先级。
` JVM`提供了10个线程优先级，但与常见的操作系统都不能很好的映射。如果希望程序能移植到各个操作系统中，应该仅仅使用`Thread`类有以下三个静态常量作为优先级，这样能保证同样的优先级采用了同样的调度方式。
 2、**线程睡眠：**`Thread.sleep(long millis)`方法，使线程转到阻塞状态。`millis`参数设定睡眠的时间，以毫秒为单位。当睡眠结束后，就转为就绪（`Runnable`）状态。`sleep()`平台移植性好。
 3、**线程等待：**`Object`类中的`wait()`方法，导致当前的线程等待，直到其他线程调用此对象的 `notify()` 方法或` notifyAll() `唤醒方法。这个两个唤醒方法也是`Object`类中的方法，行为等价于调用 wait(0) 一样。
 4、**线程让步：**`Thread.yield() `方法，暂停当前正在执行的线程对象，把执行机会让给相同或者更高优先级的线程。
 5、**线程加入：**`join()`方法，等待其他线程终止。在当前线程中调用另一个线程的`join()`方法，则当前线程转入阻塞状态，直到另一个进程运行结束，当前线程再由阻塞转为就绪状态。
 6、**线程唤醒：**Object类中的`notify()`方法，唤醒在此对象监视器上等待的单个线程。如果所有线程都在此对象上等待，则会选择唤醒其中一个线程。选择是任意性的，并在对实现做出决定时发生。线程通过调用其中一个 wait 方法，在对象的监视器上等待。 直到当前的线程放弃此对象上的锁定，才能继续执行被唤醒的线程。被唤醒的线程将以常规方式与在该对象上主动同步的其他所有线程进行竞争；例如，唤醒的线程在作为锁定此对象的下一个线程方面没有可靠的特权或劣势。类似的方法还有一个`notifyAll()`，唤醒在此对象监视器上等待的所有线程。
 注意：Thread中suspend()和resume()两个方法在JDK1.5中已经废除，不再介绍。因为有死锁倾向。

#### 常用函数说明

**1：`sleep(long millis)`: 在指定的毫秒数内让当前正在执行的线程休眠(暂停执行);**

sleep()使当前线程进入停滞状态（阻塞当前线程），让出CUP的使用、目的是不让当前线程独自霸占该进程所获的CPU资源，以留一定时间给其他线程执行的机会;
 　　 sleep()是Thread类的Static(静态)的方法；因此他不能改变对象的机锁，所以当在一个Synchronized块中调用Sleep()方法是，线程虽然休眠了，但是对象的机锁并木有被释放，其他线程无法访问这个对象（即使睡着也持有对象锁）。
 　　在sleep()休眠时间期满后，该线程不一定会立即执行，这是因为其它线程可能正在运行而且没有被调度为放弃执行，除非此线程具有更高的优先级。

**2：`join()`:指等待t线程终止。**

```java 
Thread t = new AThread(); t.start(); t.join();  
```

为什么要用join()方法

在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。

**不加`join()`方法：**

```java 
class Thread1 extends Thread{  
    private String name;  
    public Thread1(String name) {  
        super(name);  
       this.name=name;  
    }  
    public void run() {  
        System.out.println(Thread.currentThread().getName() + " 线程运行开始!");  
        for (int i = 0; i < 5; i++) {  
            System.out.println("子线程"+name + "运行 : " + i);  
            try {  
                sleep((int) Math.random() * 10);  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        }  
        System.out.println(Thread.currentThread().getName() + " 线程运行结束!");  
    }  
}  
  
public class Main {  
  
    public static void main(String[] args) {  
        System.out.println(Thread.currentThread().getName()+"主线程运行开始!");  
        Thread1 mTh1=new Thread1("A");  
        Thread1 mTh2=new Thread1("B");  
        mTh1.start();  
        mTh2.start();  
        System.out.println(Thread.currentThread().getName()+ "主线程运行结束!");  
  
    }  
  
}  
```

**输出结果：**

```java 
main主线程运行开始!
main主线程运行结束!
B 线程运行开始!
子线程B运行 : 0
A 线程运行开始!
子线程A运行 : 0
子线程B运行 : 1
子线程A运行 : 1
子线程A运行 : 2
子线程A运行 : 3
子线程A运行 : 4
A 线程运行结束!
子线程B运行 : 2
子线程B运行 : 3
子线程B运行 : 4
B 线程运行结束!
```

发现了`main`函数主线程比A，B子线程都提前结束。

**加入`join()`方法：**

(线程方法一致，不再重复)

```Java
public class Main {  
  
    public static void main(String[] args) {  
        System.out.println(Thread.currentThread().getName()+"主线程运行开始!");  
        Thread1 mTh1=new Thread1("A");  
        Thread1 mTh2=new Thread1("B");  
        mTh1.start();  
        mTh2.start();  
        try {  
            mTh1.join();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        try {  
            mTh2.join();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
        System.out.println(Thread.currentThread().getName()+ "主线程运行结束!");  
  
    }  

}  
```

**运行结果：**

```Java
main主线程运行开始!
A 线程运行开始!
子线程A运行 : 0
B 线程运行开始!
子线程B运行 : 0
子线程A运行 : 1
子线程B运行 : 1
子线程A运行 : 2
子线程B运行 : 2
子线程A运行 : 3
子线程B运行 : 3
子线程A运行 : 4
子线程B运行 : 4
A 线程运行结束!
main主线程运行结束!
```

主线程一定会等子线程都结束了才结束

**3：`yield():`暂停当前正在执行的线程对象，并执行其他线程。**

`Thread.yield()`方法作用是：暂停当前正在执行的线程对象，并执行其他线程。
` yield()`应该做的是让当前运行线程回到可运行状态，以允许具有相同优先级的其他线程获得运行机会。因此，使用yield()的目的是让相同优先级的线程之间能适当的轮转执行。但是，实际中无法保证`yield()`达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。

结论：`yield()`从未导致线程转到等待/睡眠/阻塞状态。在大多数情况下，`yield()`将导致线程从运行状态转到可运行状态，但有可能没有效果。可看上面的图。

```java
class ThreadYield extends Thread{  
    public ThreadYield(String name) {  
        super(name);  
    }  
   
    @Override  
    public void run() {  
        for (int i = 1; i <= 50; i++) {  
            System.out.println("" + this.getName() + "-----" + i);  
            // 当i为30时，该线程就会把CPU时间让掉，让其他或者自己的线程执行（也就是谁先抢到谁执行）  
            if (i ==30) {  
                this.yield();  
            }  
        }  
      
}  
}  
  
public class Main {  
  
    public static void main(String[] args) {  
          
        ThreadYield yt1 = new ThreadYield("张三");  
        ThreadYield yt2 = new ThreadYield("李四");  
        yt1.start();  
        yt2.start();  
    }  
  
}  
```

**运行结果：**

第一种情况：李四（线程）当执行到30时会CPU时间让掉，这时张三（线程）抢到CPU时间并执行。

第二种情况：李四（线程）当执行到30时会CPU时间让掉，这时李四（线程）抢到CPU时间并执行。

**sleep()和yield()的区别**

sleep()使当前线程进入停滞状态，确切来说进入阻塞状态，等sleep()规定的时间过了之后，该线程会继续执行，而停滞时间内会执行其他线程，yield()方法是直接停止该线程然后让线程从运行状态变成就绪状态，跟其他线程一块去抢夺CPU资源，有可能他会立即又抢夺到CPU资源，继续执行线程。

sleep 方法使当前运行中的线程睡眠一段时间，进入不可运行状态，这段时间的长短是由程序设定的，yield 方法使当前线程让出 CPU 占有权，但让出的时间是不可设定的。实际上，yield()方法对应了如下操作：先检测当前是否有相同优先级的线程处于同可运行状态，如有，则把 CPU  的占有权交给此线程，否则，继续运行原来的线程。所以yield()方法称为“退让”，它把运行机会让给了同等优先级的其他线程
 另外，sleep 方法允许较低优先级的线程获得运行机会，但 yield()  方法执行时，当前线程仍处在可运行状态，所以，不可能让出较低优先级的线程些时获得 CPU 占有权。在一个运行系统中，如果较高优先级的线程没有调用 sleep 方法，又没有受到 I\O 阻塞，那么，较低优先级线程只能等待所有较高优先级的线程运行结束，才有机会运行。

**4：`setPriority()`: 更改线程的优先级。**
　　MIN_PRIORITY = 1
　　NORM_PRIORITY = 5
	MAX_PRIORITY = 10

**5：`interrupt()`:**

`interrupt()`方法不是中断某个线程，而是给线程发送一个中断信号，让线程在无限等待时（如死锁时）能抛出异常，从而结束线程，但是如果你吃掉了这个异常，那么这个线程还是不会中断的！

(中断这块我会专门写一篇来讲`interrupt，isInterrupted，interrupted`。还有已经被淘汰的`stop，suspend`方法为什么会被淘汰)

**6：其他方法**

还有`wait()，notify(),notifyAll()`这些方法，因为这三个方法要跟线程的锁结合起来讲解，所以我们放在下次跟多线程的锁一块讲解。还有就是Java线程池的概念以及锁中的区别等等。

#### 线程数据传递

在传统的同步开发模式下，当我们调用一个函数时，通过这个函数的参数将数据传入，并通过这个函数的返回值来返回最终的计算结果。但在多线程的异步开发模式下，数据的传递和返回和同步开发模式有很大的区别。由于线程的运行和结束是不可预料的，因此，在传递和返回数据时就无法象函数一样通过函数参数和return语句来返回数据。

**1：通过构造方法传递数据**
 在创建线程时，必须要建立一个Thread类的或其子类的实例。因此，我们不难想到在调用start方法之前通过线程类的构造方法将数据传入线程。并将传入的数据使用类变量保存起来，以便线程使用(其实就是在run方法中使用)。下面的代码演示了如何通过构造方法来传递数据：

```Java
package mythread;   
public class MyThread1 extends Thread   
{   
private String name;   
public MyThread1(String name)   
{   
this.name = name;   
}   
public void run()   
{   
System.out.println("hello " + name);   
}   
public static void main(String[] args)   
{   
Thread thread = new MyThread1("world");   
thread.start();   
}   
}   
```

由于这种方法是在创建线程对象的同时传递数据的，因此，在线程运行之前这些数据就就已经到位了，这样就不会造成数据在线程运行后才传入的现象。如果要传递更复杂的数据，可以使用集合、类等数据结构。使用构造方法来传递数据虽然比较安全，但如果要传递的数据比较多时，就会造成很多不便。由于Java没有默认参数，要想实现类似默认参数的效果，就得使用重载，这样不但使构造方法本身过于复杂，又会使构造方法在数量上大增。因此，要想避免这种情况，就得通过类方法或类变量来传递数据。
 **2：通过变量和方法传递数据**
 向对象中传入数据一般有两次机会，第一次机会是在建立对象时通过构造方法将数据传入，另外一次机会就是在类中定义一系列的public的方法或变量（也可称之为字段）。然后在建立完对象后，通过对象实例逐个赋值。下面的代码是对MyThread1类的改版，使用了一个setName方法来设置 name变量：

```Java
package mythread;   
public class MyThread2 implements Runnable   
{   
private String name;   
public void setName(String name)   
{   
this.name = name;   
}   
public void run()   
{   
System.out.println("hello " + name);   
}   
public static void main(String[] args)   
{   
MyThread2 myThread = new MyThread2();   
myThread.setName("world");   
Thread thread = new Thread(myThread);   
thread.start();   
}   
}   
```

**3：通过回调函数传递数据**
 上面讨论的两种向线程中传递数据的方法是最常用的。但这两种方法都是main方法中主动将数据传入线程类的。这对于线程来说，是被动接收这些数据的。然而，在有些应用中需要在线程运行的过程中动态地获取数据，如在下面代码的run方法中产生了3个随机数，然后通过Work类的process方法求这三个随机数的和，并通过Data类的value将结果返回。从这个例子可以看出，在返回value之前，必须要得到三个随机数。也就是说，这个 value是无法事先就传入线程类的。

```Java
package mythread;   
class Data   
{   
public int value = 0;   
}   
class Work   
{   
public void process(Data data, Integer numbers)   
{   
for (int n : numbers)   
{   
data.value += n;   
}   
}   
}   
public class MyThread3 extends Thread   
{   
private Work work;   
public MyThread3(Work work)   
{   
this.work = work;   
}   
public void run()   
{   
java.util.Random random = new java.util.Random();   
Data data = new Data();   
int n1 = random.nextInt(1000);   
int n2 = random.nextInt(2000);   
int n3 = random.nextInt(3000);   
work.process(data, n1, n2, n3); // 使用回调函数   
System.out.println(String.valueOf(n1) + "+" + String.valueOf(n2) + "+"   
+ String.valueOf(n3) + "=" + data.value);   
}   
public static void main(String[] args)   
{   
Thread thread = new MyThread3(new Work());   
thread.start();   
}   
}   
```

#### 总结

这篇基本讲了Java多线程中的基础部分，后续还会有线程同步(锁)，线程如何正确的中断，线程池等。Java的多线程部分是比较复杂的，只有平时多看多练才能记住并应用到实际项目中去。互勉~