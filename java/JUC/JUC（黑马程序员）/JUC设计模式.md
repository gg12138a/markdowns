# 终止模式之两阶段终止模式

>Two Phase Termination

在一个线程 T1 中如何“优雅”终止线程 T2？这里的【优雅】指的是给 T2 一个料理后事的机会。





## 错误思路

- 使用线程对象的 stop() 方法停止线程：

  stop 方法会***强制杀死线程***，如果这时线程锁住了共享资源，那么当它被杀死后就再也没有机会释放锁，其它线程将永远无法获取锁

- 使用 System.exit(int) 方法停止线程：

  目的仅是停止一个线程，但这种做法***会让整个程序（进程）都停止***



## 两阶段终止模式

场景：实时监控PC状态信息

```mermaid
graph TD
w("while(true)") --> a
a("有没有被打断?") -- 是 --> b(料理后事)
b --> c(结束循环)
a -- 否 --> d(睡眠2s)
d -- 无异常 --> e(执行监控记录)
d -- 有异常 --> i(设置打断标记)
e -->w
i -->w

```





### 利用isInterrupted实现

```java
package com.example.juc_learn;

import lombok.extern.slf4j.Slf4j;

public class Test8 {
    public static void main(String[] args) throws InterruptedException {
        TwoPhaseTermination tpt = new TwoPhaseTermination();

        tpt.start();
        Thread.sleep(5_000);
        tpt.stop();

        //20:17:00.926 [Thread-0] DEBUG monitor - 记录监控信息
        //20:17:02.940 [Thread-0] DEBUG monitor - 记录监控信息
        //java.lang.InterruptedException: sleep interrupted
        //	at java.lang.Thread.sleep(Native Method)
        //	at com.example.juc_learn.TwoPhaseTermination.lambda$start$0(Test8.java:33)
        //	at java.lang.Thread.run(Thread.java:750)
        //20:17:03.917 [Thread-0] DEBUG monitor - 线程被打断，将要退出
    }
}

@Slf4j(topic = "monitor")
class TwoPhaseTermination {
    private Thread monitor;  //用于监控信息的线程

    //启动monitor线程
    public void start() {
        monitor = new Thread(() -> {
            while (true) {
                Thread current = Thread.currentThread();
                if (current.isInterrupted()) {

                    //此线程被打断，进行后续处理
                    log.debug("线程被打断，将要退出");
                    break;
                } else {
                    try {
                        Thread.sleep(2_000);

                        //进行信息监控及记录
                        log.debug("记录监控信息");
                    } catch (InterruptedException e) {

                        //重新设置打断标记
                        current.interrupt();
                        e.printStackTrace();
                    }
                }
            }
        });

        monitor.start();
    }

    //结束monitor线程
    public void stop() {
        monitor.interrupt();
    }
}
```



说明：关于interrupt()

- 若被interrupt的线程，处于非阻塞状态，将设置打断标记为true
- 否则，将清除打断标记并抛出异常。

因此，有如下情况：

- 当monitor线程正常运行时被打断，将设置打断标记为true

- 当monitor线程处于sleep时被打断，打断标记将被清除，并抛出InterruptedException异常，从而进入catch块，将再次调用interrupt方法，设置打断标记为true

  当检测到当前线程的打断标记为真时，进行后续处理操作，安全结束线程



### 利用volitile实现

```java
public class Test8 {
    public static void main(String[] args) throws InterruptedException {
        TwoPhaseTermination tpt = new TwoPhaseTermination();

        tpt.start();
        Thread.sleep(5_000);
        tpt.stop();
    }
}

@Slf4j(topic = "c.monitor")
class TwoPhaseTermination {
    private Thread monitor;
    private volatile boolean stop;

    public void start() {
        monitor = new Thread(() -> {
            while (true) {
                if (stop) {
                    log.debug("线程被打断，将要退出");
                    break;
                }

                try {
                    Thread.sleep(2_000);
                    log.debug("记录监控信息");
                } catch (InterruptedException e) {

                    stop = true;    
                    e.printStackTrace();
                }
            }
        });

        monitor.start();
    }

    public void stop() {
        stop = true;
                monitor.interrupt();	// 考虑stop为true，但正在sleep的情况。
    }
}
```







# 同步模式之保护性暂停

## 定义

即 Guarded Suspension，用在一个线程等待另一个线程的执行结果

要点：

- 有**<u>*一个结果*</u>**需要从一个线程传递到另一个线程，让他们关联同一个 GuardedObject 

  > 但如果有结果不断从一个线程到另一个线程，则应采用消息队列（见生产者/消费者）

- JDK 中，join 的实现、Future 的实现采用的就是此模式

> 因为要等待另一方的结果，因此归类到同步模式



![image-20220427200712055](JUC%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.assets/image-20220427200712055.png)



## 实现

```java
@Slf4j(topic = "c.Test20")
public class Test20 {

    public static void main(String[] args) {
        // 模拟 线程1 等待 线程2

        GuardedObject guardedObject = new GuardedObject();

        new Thread(() -> {
            List<String> list = (List<String>) guardedObject.get();
            log.debug("等待结束。结果长度：{}",list.size());
        }, "thread1").start();

        new Thread(()->{
            try {
                log.debug("开始下载");
                List<String> download = Downloader.download();

                guardedObject.complete(download);
            } catch (IOException e) {
                e.printStackTrace();
            }
        },"thread2").start();
    }
}


class GuardedObject {

    // 即运行结果
    private Object response;

    // 获取结果
    public Object get() {
        synchronized (this) {
            while (response == null) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            return response;
        }
    }
    
    // 获取结果
    // timeout表示最长等待时间
    public Object get(long timeout) {
        synchronized (this) {
            long begin = System.currentTimeMillis();
            long passedTime = 0;

            while (response == null) {
                if (passedTime >= timeout)
                    break;

                // 需考虑虚假唤醒问题
                try {
                    this.wait(timeout - passedTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                passedTime = System.currentTimeMillis() - begin;
            }

            return response;
        }
    }
    
    // 产生结果
    public void complete(Object response) {
        synchronized (this) {
            this.response = response;
            this.notifyAll();
        }
    }
}
```



## 拓展（多任务版 GuardedObject）

图中 Futures 就好比居民楼一层的信箱（每个信箱有房间编号），左侧的 t0，t2，t4 就好比等待邮件的居民，右侧的 t1，t3，t5 就好比邮递员

如果需要在多个类之间使用 GuardedObject 对象，作为参数传递不是很方便，因此设计一个用来解耦的中间类， 这样不仅能够***解耦【结果等待者】和【结果生产者】***，还能够同时支持多个任务的管理

![image-20220427205156863](JUC%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.assets/image-20220427205156863.png)

> 区别于生产者消费者模式的特点：
>
> ***一个生产者和一个消费者之间***，是相互***一一对应的***。





- 新增 id 用来标识 Guarded Object

  ```java
  class GuardedObject {
  
      // 用于唯一表示GuardedObject
      private int id;
  
      public int getId() {
          return id;
      }
  
      public GuardedObject(int id) {
          this.id = id;
      }
  
      private Object response;
  
      public Object get(long timeout) {
          synchronized (this) {
              long begin = System.currentTimeMillis();
              long passedTime = 0;
  
              while (response == null) {
                  if (passedTime >= timeout)
                      break;
  
                  // 需考虑虚假唤醒问题
                  try {
                      this.wait(timeout - passedTime);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  passedTime = System.currentTimeMillis() - begin;
              }
  
              return response;
          }
      }
  
      public void complete(Object response) {
          synchronized (this) {
              this.response = response;
              this.notifyAll();
          }
      }
  }
  ```

- 中间解耦类

  ```java
  class MailBoxes {
      private static Map<Integer, GuardedObject> boxes = new Hashtable<>();
      private static int id = 1;
  
      public static synchronized int generateId() {
          return id++;
      }
  
      public static GuardedObject createGuardObject() {
          GuardedObject go = new GuardedObject(generateId());
          boxes.put(go.getId(), go);
          return go;
      }
  
      public static GuardedObject getGuardObject(int id) {
          return boxes.remove(id);
      }
  
      public static Set<Integer> getIds() {
          return boxes.keySet();
      }
  }
  ```

- 业务相关类

  ```java
  // 居民类
  @Slf4j(topic = "c.People")
  class People extends Thread {
  
      @Override
      public void run() {
          GuardedObject guardObject = MailBoxes.createGuardObject();
          Object mail = guardObject.get(5000);
          log.debug("id:{},receivedMail：{}", guardObject.getId(), mail);
      }
  }
  ```

  ```java
  // 邮递员类
  @Slf4j(topic = "c.Postman")
  class Postman extends Thread {
      private int mailBoxId;
      private String mail;
  
      public Postman(int mailBoxId, String mail) {
          this.mailBoxId = mailBoxId;
          this.mail = mail;
      }
  
      @Override
      public void run() {
          GuardedObject guardObject = MailBoxes.getGuardObject(this.mailBoxId);
          guardObject.complete(this.mail);
          log.debug("id:{},deliverMail:{}", guardObject.getId(), mail);
      }
  }
  ```

- 测试

  ```java
  @Slf4j(topic = "c.Test20")
  public class Test20 {
  
      public static void main(String[] args) throws InterruptedException {
          for (int i = 0; i < 3; i++) {
              new People().start();
          }
          Thread.sleep(1000);
  
          for (Integer id : MailBoxes.getIds()) {
              // 注意：一个生产者和一个消费者相互对应
              new Postman(id, "content" + id).start();
          }
      }
  }
  ```

  ```
  21:14:42 [Thread-4] c.Postman - id:2,deliverMail:content2
  21:14:42 [Thread-2] c.People - id:2,receivedMail：content2
  21:14:42 [Thread-3] c.Postman - id:3,deliverMail:content3
  21:14:42 [Thread-1] c.People - id:3,receivedMail：content3
  21:14:42 [Thread-5] c.Postman - id:1,deliverMail:content1
  21:14:42 [Thread-0] c.People - id:1,receivedMail：content1
  ```




# 异步模式之生产者/消费者

## 定义

> #### 💡 为什么是异步模式？
>
> 因为生产者的生产结果，并不一定立即被消费

> 区别与多任务版的GuradedObjcet：
>
> 生产者消费者模式，***不需要生成结果的线程和消费结果的线程，一一对应***。

- ***消费队列可以用来平衡生产和消费的线程资源***
- ***生产者仅负责产生结果数据***，不关心数据该如何处理，而***消费者专心处理结果数据***

- 消息队列是有容量限制的，满时不会再加入数据，空时不会再消耗数据

- JDK 中各种阻塞队列，采用的就是这种模式



例如如下的情况：

![image-20220427212225651](JUC%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.assets/image-20220427212225651.png)



## 实现

- 消息载体类：

  ```java
  final class Message {
      private int id;
      private Object value;
  
      public Message(int id, Object value) {
          this.id = id;
          this.value = value;
      }
  
      public int getId() {
          return id;
      }
  
      public Object getValue() {
          return value;
      }
  
      @Override
      public String toString() {
          return "Message{" +
                  "id=" + id +
                  ", value=" + value +
                  '}';
      }
  }
  ```

- 消息队列（仅能在Java**<u>线程间</u>**通信）：

  ```java
  // Java线程间 通信的消息队列
  @Slf4j(topic = "c.MessageQueue")
  class MessageQueue {
  
      private final LinkedList<Message> list = new LinkedList<>();
      private final int capacity;
  
      public MessageQueue(int capacity) {
          this.capacity = capacity;
      }
  
      public Message take() {
          synchronized (list) {
              while (list.isEmpty()) {
                  try {
                      log.debug("队列为空,消费者线程进入等待");
                      list.wait();
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
  
              Message message = list.removeFirst();
              log.debug("消费：{}", message);
              list.notifyAll();
              return message;
          }
      }
  
      public void put(Message message) {
          synchronized (list) {
              while (list.size() == this.capacity) {
                  try {
                      log.debug("队列为满，生产者线程进入等待");
                      list.wait();
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
  
              list.addLast(message);
              log.debug("生产：{}", message);
              list.notifyAll();
          }
      }
  }
  ```

- 测试类：

  ```java
  @Slf4j(topic = "c.Test21")
  public class Test21 {
  
      public static void main(String[] args) throws InterruptedException {
          MessageQueue queue = new MessageQueue(2);
  
          // 生产者线程
          for (int i = 0; i < 3; i++) {
              int id = i;
              new Thread(() -> {
                  queue.put(new Message(id, "message-" + id));
              }, "producer-" + i).start();
          }
  
          Thread.sleep(1000);
  
          // 消费者线程
          new Thread(() -> {
              while (true) {
                  Message message = queue.take();
              }
          }, "consumer").start();
      }
  
  }
  ```

  ```
  22:03:28 [producer-0] c.MessageQueue - 生产：Message{id=0, value=message-0}
  22:03:28 [producer-2] c.MessageQueue - 生产：Message{id=2, value=message-2}
  22:03:28 [producer-1] c.MessageQueue - 队列为满，生产者线程进入等待
  22:03:29 [consumer] c.MessageQueue - 消费：Message{id=0, value=message-0}
  22:03:29 [consumer] c.MessageQueue - 消费：Message{id=2, value=message-2}
  22:03:29 [consumer] c.MessageQueue - 队列为空,消费者线程进入等待
  22:03:29 [producer-1] c.MessageQueue - 生产：Message{id=1, value=message-1}
  22:03:29 [consumer] c.MessageQueue - 消费：Message{id=1, value=message-1}
  22:03:29 [consumer] c.MessageQueue - 队列为空,消费者线程进入等待
  ```






# 同步模式之顺序控制



## 固定运行顺序

比如，存在一种需求，必须先 2 再1 。



### wait-notify实现

```java
@Slf4j(topic = "c.Test25")
public class Test25 {

    private static final Object obj = new Object();
    private static boolean done = false;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (obj) {
                while (!done) {
                    try {
                        obj.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                log.debug("1");
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            synchronized (obj) {
                done = true;
                log.debug("2");

                obj.notifyAll();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```



### ReentrantLock(await-signalAll)实现

```java
@Slf4j(topic = "c.Test26")
public class Test26 {

    private static final ReentrantLock lock = new ReentrantLock();
    private static boolean done = false;

    public static void main(String[] args) {
        Condition isDone = lock.newCondition();

        Thread t1 = new Thread(() -> {
            lock.lock();
            try {
                while (!done) {
                    try {
                        isDone.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                log.debug("1");
            } finally {
                lock.unlock();
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            lock.lock();
            try {
                done = true;
                log.debug("2");

                isDone.signalAll();
            } finally {
                lock.unlock();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```



### LockSupport(park-unpark)实现

```java
@Slf4j(topic = "c.Test27")
public class Test27 {

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            LockSupport.park();
            log.debug("1");
        }, "t1");

        Thread t2 = new Thread(() -> {
            log.debug("2");
            LockSupport.unpark(t1);
        }, "t2");

        t1.start();
        t2.start();
    }
}
```



## 交替运行顺序

线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。现在要求输出 abcabcabcabcabc 怎么实现



### wait-notify实现

```java
@Slf4j(topic = "c.Test28")
public class Test28 {

    public static void main(String[] args) {
        FlagObject flagObject = new FlagObject(1, 5);

        new Thread(() -> {
            flagObject.print("a", 1, 2);
        }, "t1").start();

        new Thread(() -> {
            flagObject.print("b", 2, 3);
        }, "t2").start();

        new Thread(() -> {
            flagObject.print("c", 3, 1);
        }, "t3").start();
    }
}

@Slf4j(topic = "c.FlagObject")
class FlagObject {
    private int flag;

    private int loopNumber;

    public FlagObject(int flag, int loopNumber) {
        this.flag = flag;
        this.loopNumber = loopNumber;
    }

    public synchronized void print(String str, int waitFlag, int nextFlag) {
        for (int i = 0; i < loopNumber; i++) {
            while (flag != waitFlag) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            log.debug("{}", str);

            flag = nextFlag;
            this.notifyAll();
        }
    }
}
```



### ReentrantLock实现

```java
public class Test29 {

    public static void main(String[] args) throws InterruptedException {
        AwaitSignal awaitSignal = new AwaitSignal(5);
        Condition conditionA = awaitSignal.newCondition();
        Condition conditionB = awaitSignal.newCondition();
        Condition conditionC = awaitSignal.newCondition();

        Thread stater = new Thread(() -> {
            LockSupport.park();
            awaitSignal.lock();
            try {
                conditionA.signalAll();
            } finally {
                awaitSignal.unlock();
            }
        }, "stater");

        stater.start();


        new Thread(() -> {
            awaitSignal.print(conditionA, "a", conditionB, stater);
        }, "t1").start();

        new Thread(() -> {
            awaitSignal.print(conditionB, "b", conditionC, stater);
        }, "t2").start();

        new Thread(() -> {
            awaitSignal.print(conditionC, "c", conditionA, stater);
        }, "t3").start();

    }
}

@Slf4j(topic = "c.AwaitSignal")
class AwaitSignal extends ReentrantLock {
    private int loopNumber;
    private int lockedNum = 0;

    public int getLockedNum() {
        return lockedNum;
    }

    public AwaitSignal(int loopNumber) {
        this.loopNumber = loopNumber;
    }

    public void print(Condition current, String msg, Condition next, Thread thread) {
        for (int i = 0; i < loopNumber; i++) {
            this.lock();
            if (++lockedNum == 3) {
                LockSupport.unpark(thread);
            }

            try {
                try {
                    current.await();
                    log.debug("{}", msg);
                    next.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } finally {
                this.lockedNum--;
                this.unlock();
            }
        }
    }
}
```



### LockSupport实现

```java
@Slf4j(topic = "c.Test30")
public class Test30 {

    private static Thread t1;
    private static Thread t2;
    private static Thread t3;

    public static void main(String[] args) {
        ParkUnpark pu = new ParkUnpark(5);

        Thread starter = new Thread(() -> {
            LockSupport.park();
            LockSupport.unpark(t1);
        }, "starter");

        t1 = new Thread(() -> {
            pu.print("a", t2, starter);
        }, "t1");

        t2 = new Thread(() -> {
            pu.print("b", t3, starter);
        }, "t2");

        t3 = new Thread(() -> {
            pu.print("c", t1, starter);
        }, "t3");

        t1.start();
        t2.start();
        t3.start();
        starter.start();
    }
}

@Slf4j(topic = "c.ParkUnpark")
class ParkUnpark {
    private int loopNumber;
    private int parked = 0;
    private boolean started;

    public ParkUnpark(int loopNumber) {
        this.loopNumber = loopNumber;
    }

    public void print(String msg, Thread next, Thread starter) {
        for (int i = 0; i < loopNumber; i++) {
            if (!started) {
                synchronized (this) {
                    if (++parked == 3) {
                        LockSupport.unpark(starter);
                        started = true;
                    }
                }
            }

            LockSupport.park();
            log.debug("{}", msg);
            LockSupport.unpark(next);
        }
    }
}
```





# 同步模式之犹豫模式

## 定义

Balking （犹豫）模式用在一个线程发现另一个线程或本线程已经做了某一件相同的事，那么本线程就无需再做了，直接结束返回

> 对比一下保护性暂停模式：保护性暂停模式用在一个线程等待另一个线程的执行结果，当条件不满足时线程等待。



## 实现

```java
public class BalkingTest {

    public static void main(String[] args) throws InterruptedException {
        StatusMonitor statusMonitor = new StatusMonitor();

        statusMonitor.start();
        statusMonitor.start();
        statusMonitor.start();
        Thread.sleep(5000);
        statusMonitor.stop();
    }
}

@Slf4j(topic = "c.monitor")
class StatusMonitor {
    private Thread monitor;
    private volatile boolean stop;
    private volatile boolean started;

    public void start() {
        if (!started) {

            synchronized (this) {
                if (started)
                    return;

                monitor = new Thread(() -> {
                    while (true) {
                        if (stop) {
                            log.debug("线程被打断，将要退出");
                            break;
                        }

                        try {
                            Thread.sleep(2_000);
                            log.debug("记录监控信息");
                        } catch (InterruptedException e) {

                            stop = true;
                            e.printStackTrace();
                        }
                    }
                });
                monitor.start();

                started = true;
            }
        }
        
        // 若已启动过，则直接返回。即Balking
    }

    public void stop() {
        stop = true;
        monitor.interrupt();
    }
}
```



# 享元模式

## 定义

Flyweight pattern，用于**重用数量有限的同一类对象**，最小化内存的使用。



## JDK中的体现

在JDK中 Boolean，Byte，Short，Integer，Long，Character 等包装类提供了 valueOf 方法，例如 Long 的 valueOf 会缓存 -128~127 之间的 Long 对象，在这个范围之间会重用对象，大于这个范围，才会新建 Long 对 象：

```java
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

>注意： 
>
>- Byte, Short, Long 缓存的范围都是 -128~127
>
>- Character 缓存的范围是 0~127
>
>- Integer的默认范围是 -128~127 
>
>  - 最小值不能变 
>
>  - 但最大值可以通过调整虚拟机参数:
>
>     `  -Djava.lang.Integer.IntegerCache.high` 来改变 
>
>- Boolean 缓存了 TRUE 和 FALSE



类似的案例，还有：

1. String 串池
2. BigDecimal、BigInteger等



## 案例——数据库连接池

```java
@Slf4j(topic = "c.Pool")
public class Pool {
    private final int poolSize;
    private Connection[] connections;
    private AtomicIntegerArray status;

    public Pool(int poolSize) {
        this.poolSize = poolSize;
        this.connections = new Connection[poolSize];
        this.status = new AtomicIntegerArray(new int[poolSize]);

        for (int i = 0; i < poolSize; i++) {
            connections[i] = new MockConnection();
        }
    }

    public Connection borrow() {
        while (true) {
            for (int i = 0; i < poolSize; i++) {
                if (status.get(i) == 0) {
                    if (status.compareAndSet(i, 0, 1)) {

                        log.debug("获取连接：{}", i);
                        return connections[i];
                    }
                }
            }

            // 当前无空闲连接
            synchronized (this) {
                try {
                    log.debug("当前无空闲连接，进入阻塞等待");
                    this.wait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    public void freeConnection(Connection conn) {
        for (int i = 0; i < poolSize; i++) {
            if (conn == connections[i]) {
                status.set(i, 0);   // 连接持有者只会同时有一个，可以不用CAS操作

                log.debug("归还连接：{}", i);
                synchronized (this) {
                    this.notifyAll();
                }

                break;
            }
        }
    }

    public static void main(String[] args) {
        Pool pool = new Pool(2);

        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                Connection connection = pool.borrow();

                try {
                    Thread.sleep(new Random().nextInt(1000));
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                pool.freeConnection(connection);
            }).start();
        }
    }
}

class MockConnection implements Connection {
    ...（空
}
```



# 异步模式之工作线程

## 定义

让有限的工作线程（Worker Thread）来轮流异步处理无限多的任务。也可以将其归类为<u>**分工模式**</u>，它的典型实现 就是线程池，也体现了经典设计模式中的<u>**享元模式**</u>。

> 注意：不同任务类型，应该使用不同的线程池，以避免饥饿。



## 饥饿

> 此处的饥饿现象，是由于**线程数目不足**而引起的。
>
> #### 💡 固定大小线程池会有饥饿现象



两个工人是同一个线程池中的两个线程：

- 他们要做的事情是：为客人点餐和到后厨做菜，这是两个阶段的工作：
  - 客人点餐：必须先点完餐，等菜做好，上菜，在此期间处理点餐的工人**必须等待**
  - 后厨做菜
- 比如工人A 处理了点餐任务，接下来它要等着 工人B 把菜做好，然后上菜，他俩也配合的蛮好
- 但现在同时来了两个客人，这个时候工人A 和工人B 都去处理点餐了，这时没人做饭了，**产生饥饿**

```java
public class TestDeadLock {
    static final List<String> MENU = Arrays.asList("地三鲜", "宫保鸡丁", "辣子鸡丁", "烤鸡翅");
    static Random RANDOM = new Random();
    static String cooking() {
        return MENU.get(RANDOM.nextInt(MENU.size()));
    }
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.execute(() -> {
            log.debug("处理点餐...");
            Future<String> f = executorService.submit(() -> {
                log.debug("做菜");
                return cooking();
            });
            try {
                log.debug("上菜: {}", f.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        });

        // 若再提交一个此任务，则会因线程数不够而导致饥饿
        executorService.execute(() -> {
            log.debug("处理点餐...");
            Future<String> f = executorService.submit(() -> {
                log.debug("做菜");
                return cooking();
            });
            try {
                log.debug("上菜: {}", f.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        });
    }
}
```

> 解决方案：将不同任务，提交到不同的线程池



## 创建多少线程比较合适

-  过少：程序不能充分的利用系统资源
- 过多：导致更多的上下文切换；占用更多内存



结论：

- CPU密集型任务：

  通常采用 cpu 核数 + 1 能够实现最优的 CPU 利用率

  > +1 是保证当线程由于页缺失故障（操作系统）或其它原因导致暂停时，额外的这个线程就能顶上去，保证 CPU 时钟周期不被浪费

- I/O 密集型任务：

  CPU 不总是处于繁忙状态，例如，当你执行业务计算时，这时候会使用 CPU 资源，但当你执行 I/O 操作时、远程 RPC 调用时，包括进行数据库操作时，这时候 CPU 就闲下来了，你可以利用多线程提高它的利用率。

  经验公式如下：

  `线程数 = 核数 * 期望 CPU 利用率 * 总时间(CPU计算时间+等待时间) / CPU 计算时间`

  例如 4 核 CPU 计算时间是 50% ，其它等待时间是 50%，期望 cpu 被 100% 利用，套用公式 4 * 100% * 100% / 50% = 8
