# 话说 wait、notify 、 notifyAll



# 一、前言

说起java的线程之间的通信，难免会想起它，他就是 wait 、notify、notifyAll

1. 他们三个都是Object类的方法， 受到 final  和  native 加持 ，也就造就了他们是不能被重写的 
2. wait() 等待 ，意味让出当前线程的锁，进入等待状态，让其他线程先用会儿锁 ，这里注意了，什么叫让出当前线程的锁？ 也就是你当前线程必须要先获得锁，所以它一般会与synchronized（我的上一篇文章有写）配合使用
   官方注释： *The current thread must own this object's monitor.* 
   wait要抛出InterruptedException异常 需要try catch 因为线程wait期间可能会被打断。
3. notify()  唤醒一个wait()的线程，当notify所在的代码块的锁释放之后，wait的线程开始抢锁，嗯....... ,Object类里注释写的是唤醒wait线程是任意(arbitrary)的 ,但是可以由具体实现自行裁决，我看hotspot实现好像是用的双向链表，notify的时候是从head拿出一个唤醒，所以我称之为有序,如果有问题请读者指出。 
4. notifyAll () 唤醒所有wait线程，notify的高级版本  
5. 注意事项： 并不是说notify之后 wait的线程就能马上执行，因为wait是放弃了当前线程的锁，被notify之后还需要自己去抢锁，如果notify所在的代码块还没有抢到锁，或者被其他线程把锁抢到了，那wait所在线程还需要接着努力抢锁。

# 二、DEMO 

###  1.wait notify 简单使用   
是这样的, 小明做了饭，给二月鸟吃(备注：二月鸟 是一个人名)，只有一双筷子， 小明需要先尝一口看能不能吃， 然后再通知二月鸟吃饭，二月鸟要等小明放下筷子才能拿起筷子吃饭。用程序怎么实现，实现方式很多  咱今天只论wait notify 其他方式靠边儿站

```java
public class WaitNotifyTest {
    public static void main(String[] args)   {
        // 这是一把锁  筷子
        Object obj = new Object();

        new Thread(()->{
            synchronized (obj){
                try {
                    System.out.println("二月鸟来了 等着吃饭...");
                    obj.wait();
                    System.out.println("二月鸟拿到筷子吃饭喽...");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        new Thread(()->{
            synchronized (obj){
                try{
                    System.out.println("小明尝一下可以吃，通知大家吃饭...");
                    // 通知二月鸟 可以吃饭了
                    obj.notify();
                    System.out.println("这个时候小明还没有放下筷子...");
                    TimeUnit.SECONDS.sleep(5);
                    System.out.println("小明放筷子了...");
                } catch (Exception e) {
                    System.out.println("中毒了..");
                }
            }
        }).start();
    }
}

执行结果：
二月鸟来了 等着吃饭...
小明尝一下可以吃，通知大家吃饭...
这个时候小明还没有放下筷子...
小明放筷子了...
二月鸟拿到筷子吃饭喽...
```

 这里需要注意几个点：

1. wait需要在synchronized中包裹着
2. notify需要synchronized中包裹着
3. notify之后 二月鸟没有马上拿起筷子吃饭，因为小明还没有放下筷子（锁还没释放）
4. 这个故事里，小明有点儿不地道了，他还没准备放筷子就通知二月鸟可以吃饭了，害的二月鸟等了半天，我们不能学小明，我们平时写代码，一般业务执行完了，代码块最后执行notify，执行完notify之后线程马上就会释放锁。

### 2. wait notifyAll 简单使用

还是1中的例子，小明做完饭后，二月鸟和小月月都来吃饭了,还是只有一双筷子（真穷）， 这时候我们用wait notify 试一下 大家看看 

```java
public class WaitNotifyTest02 {
    public static void main(String[] args)   {
        // 这是一把锁  筷子
        Object obj = new Object();

        new Thread(()->{
            synchronized (obj){
                try {
                    System.out.println("二月鸟来了 等着吃饭...");
                    obj.wait();
                    System.out.println("二月鸟拿到筷子吃饭喽...");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("二月鸟吃完了 放下筷子...");
                }
            }
        }).start();

        new Thread(()->{
            synchronized (obj){
                try {
                    System.out.println("小月月来了 等着吃饭...");
                    obj.wait();
                    System.out.println("小月月拿到筷子吃饭喽...");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    System.out.println("小月月吃完了 放下筷子...");
                }
            }
        }).start();

        new Thread(()->{
            synchronized (obj){
                try{
                    System.out.println("小明尝一下可以吃，通知大家吃饭...");
                    // 通知等待吃饭的一个人（注意是一个人） 可以吃饭了
                    obj.notify();
                    System.out.println("这个时候小明还没有放下筷子...");
                    TimeUnit.SECONDS.sleep(5);
                    System.out.println("小明放筷子了...");
                } catch (Exception e) {
                    System.out.println("中毒了..");
                }
            }
        }).start();
    }
}

执行结果：
二月鸟来了 等着吃饭...
小月月来了 等着吃饭...
小明尝一下可以吃，通知大家吃饭...
这个时候小明还没有放下筷子...
小明放筷子了...
二月鸟拿到筷子吃饭喽...
二月鸟吃完了 放下筷子...

```

咦？ 小月月怎么不吃饭， 二月鸟放下筷子了呀！  

1. notify 只通知一个wait线程结束wait状态  
2. 这里可以看出 hotspot实现 是按照wait的先后顺序通知的 
3. 虽然是按照顺序通知的，但是我们不能依赖这个规律，因为他仅仅是规律，在别的系统（可能安装不同的JVM实现）上不一定有这个规律

其他都不变，notify 改为notifyAll 

```java
new Thread(()->{
            synchronized (obj){
                try{
                    System.out.println("小明尝一下可以吃，通知大家吃饭...");
                    // 通知等待吃饭的人，所有人 可以吃饭了
                    obj.notifyAll();
                    System.out.println("这个时候小明还没有放下筷子...");
                    TimeUnit.SECONDS.sleep(5);
                    System.out.println("小明放筷子了...");
                } catch (Exception e) {
                    System.out.println("中毒了..");
                }
            }
        }).start();

执行结果：
二月鸟来了 等着吃饭...
小月月来了 等着吃饭...
小明尝一下可以吃，通知大家吃饭...
这个时候小明还没有放下筷子...
小明放筷子了...
小月月拿到筷子吃饭喽...
小月月吃完了 放下筷子...
二月鸟拿到筷子吃饭喽...
二月鸟吃完了 放下筷子...
```

这里可以看到： 小月月居然比二月鸟先吃到饭，这里是因为notifyAll 是唤醒了所有人，谁抢到筷子（锁），谁先吃（执行）

经过我的测试，我发现大概规律是按照wait的反向顺序来的，也就是先wait的后吃饭

# 三、假装学术讨论

### 3.1 hotspot 实现 ，notify是按wait顺序的？

<u>以下内容非虚构，但纯属个人见解，请勿当做理论来用，可作为参考</u>

#### 3.1.1 hotspot wait 代码(删减版)

```c++
// millis:wait超时时间  interruptible：是否可中断   TRAPS：调用wait的线程
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) { 
  Thread * const Self = THREAD;
  JavaThread * jt = Self->as_Java_thread();

  //  这个方法就是判断当前线程是否获得了锁，如果不在synchronized代码块就会抛异常
  //  看 check_owner方法
  CHECK_OWNER();  

  // 一堆代码 
  
  // 貌似是申请锁
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - add");
  // 这个就是把wait的线程放到一个“集合”里  看AddWaiter方法
  AddWaiter(&node);
  // 貌似是释放锁 
  Thread::SpinRelease(&_WaitSetLock);
  // 一堆代码
}
```



```c++
// Returns true if the specified thread owns the ObjectMonitor.
// 翻译：如果指定的线程拥有ObjectMonitor 也就是获得了锁  就返回true
// Otherwise returns false and throws IllegalMonitorStateException
// 翻译：否则返回false 并且抛出异常 IllegalMonitorStateException
// (IMSE). If there is a pending exception and the specified thread
// is not the owner, that exception will be replaced by the IMSE.
// 这句不会翻译了
bool ObjectMonitor::check_owner(Thread* THREAD) {
  void* cur = owner_raw();
  if (cur == THREAD) {
    return true;
  }
  if (THREAD->is_lock_owned((address)cur)) {
    set_owner_from_BasicLock(cur, THREAD);  // Convert from BasicLock* to Thread*.
    _recursions = 0;
    return true;
  }
  // 这里抛出了异常  
  THROW_MSG_(vmSymbols::java_lang_IllegalMonitorStateException(),
             "current thread is not owner", false);
}
```

抛异常示例：

```java
public class WaitNotifyTest04 {
    public static void main(String[] args)   {
        Object obj = new Object();
        new Thread(()->{
            try {
                obj.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

异常信息：
    Exception in thread "Thread-0"       
    java.lang.IllegalMonitorStateException
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at WaitNotifyTest04.lambda$main$0(WaitNotifyTest04.java:23)
	at java.lang.Thread.run(Thread.java:748)
```

```c++

inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  // put node at end of queue (circular doubly linked list)
  // 翻译： 把node放到队列的最后边（双向链表）
  // 如果你不知道什么是循环双向链表 我给你画出来
  
  // _WaitSet是头元素 其实玩儿过链表的人 这里应该都很清楚 
  // 这里所谓的头 不是真正的头 只是一个相对概念 
  // 如果是空的 那就一个waiter ,_next _prev 都指向自己
  if (_WaitSet == NULL) {
    _WaitSet = node;
    node->_prev = node;
    node->_next = node;
  } else {
      
    // 如果已经有了元素 那就把node放最后  
    // 然后node的_next 指向_WaitSet也就是头元素  
    // node的_prev指向添加之前的头元素
    ObjectWaiter* head = _WaitSet;
    ObjectWaiter* tail = head->_prev;
    assert(tail->_next == head, "invariant check");
    tail->_next = node;
    head->_prev = node;
    node->_next = head;
    node->_prev = tail;
  }
}
```

循环双向链表：

不知道大家有没有听说过<a>Disruptor</a>这个框架 ,他好像就是类似的结构  ，这个框架把性能做到了极致，有时间大家可以了解下

![循环双向链表](https://github.com/githubforliming/blog/blob/main/typora_images/waitnotify/循环双向链表.png)



#### 3.1.2 hotspot notify代码(删减版)：

```c++

void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();  // 检测是否拥有锁
  // 如果没有wait线程 直接返回 
  // 这也是为什么  一个线程先notify 另一个线程再wait 是不能唤醒的 必须是先wait的线程才能被notify
  // 
  if (_WaitSet == NULL) {
    return;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);
  // 主要看这个 
  INotify(THREAD);
  OM_PERFDATA_OP(Notifications, inc(1));
}
```

```c++
// Consider:
// If the lock is cool (cxq == null && succ == null) and we're on an MP system
// 翻译： 如果锁符合条件(cxq == null && succ == null) 并且在一个MP系统 
// 说实话 我翻译不下去了 ， 主要看下边这句 
// then instead of transferring a thread from the WaitSet to the EntryList
// 翻译：我们将从WaitSet中取一个线程到EntryList 
// EntryList 里是啥 就是唤醒的线程集合  
// we might just dequeue a thread from the WaitSet and directly unpark() it.

void ObjectMonitor::INotify(Thread * Self) {
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - notify");
  // 主要看这 DequeueWaiter就是从wiatset里取出一个 线程 
  // 怎么取？从头取 头是谁？ 第一个wiat的线程呗  看后边代码
  ObjectWaiter * iterator = DequeueWaiter();
  if (iterator != NULL) {
    // 一堆代码 
      
    // 这里需要注意 注意  注意  
    //1. 如果list是空 就把list指向iterator 也就是取出来那个元素 
    // notify 的时候 是一个 走 if 
    if (list == NULL) {
      iterator->_next = iterator->_prev = NULL;
      _EntryList = iterator;
    } else {
      // 如果不是空 注意了 注意了  notifyAll的时候 大多数会走这里 
      iterator->TState = ObjectWaiter::TS_CXQ;
      for (;;) {
        // 这里不是很懂 _cxq貌似是指向的_EntryList的第一个元素 
        ObjectWaiter * front = _cxq;
        iterator->_next = front;
        // cmpxchg : 如果&_cxq 等于  front 就把iterator写到&_cxq内存
        // 这个操作时什么呢 把头上拿出来的waiter放到_EntryList的首元素位置 ！！
        // 再品一下这句话  等会儿看notifyAll的时候 会用到这个知识
        if (Atomic::cmpxchg(&_cxq, front, iterator) == front) {
          break;
        }
      }
    }
    
    iterator->wait_reenter_begin(this);
  }
  Thread::SpinRelease(&_WaitSetLock);
}
```

```c++

inline ObjectWaiter* ObjectMonitor::DequeueWaiter() {
  // dequeue the very first waiter
  // 人家都注释了 取出第一个waiter 这就是为什么notify是按wait顺序来的 
  ObjectWaiter* waiter = _WaitSet;
  if (waiter) {
    // 看这个方法 ↓ 
    DequeueSpecificWaiter(waiter);
  }
  return waiter;
}

inline void ObjectMonitor::DequeueSpecificWaiter(ObjectWaiter* node) {
  // when the waiter has woken up because of interrupt,
  // timeout or other spurious wake-up, dequeue the
  // waiter from waiting list
  ObjectWaiter* next = node->_next;
  // 如果_next 等于自己 那说明就一个waiter 
  // 闭眼想：是不是一个的时候自己的_next是指向自己的 老光棍都懂，左手拉右手。。
  if (next == node) {
    _WaitSet = NULL;//因为就一个 拿出后把waitset置空
  } else {
    // 这波操作 就是把头结点 毫无痕迹的取出来
    // 末尾元素_next指向 第二个元素 
    // 第二个元素的_prev 指向末尾元素 
    ObjectWaiter* prev = node->_prev;
    next->_prev = prev;
    prev->_next = next;
    // 把_WaitSet引用到next 也就是第二个元素上  第一个元素拜拜了您嘞
    if (_WaitSet == node) {
      _WaitSet = next;
    }
  }
  // _next _prev 置空  留一个node 孤零零  给大家画一下看一眼 
  node->_next = NULL;
  node->_prev = NULL;
}
```

取出前：

![出队列01](https://github.com/githubforliming/blog/blob/main/typora_images/waitnotify/出队列01.png)

取出后：

![出队列02](https://github.com/githubforliming/blog/blob/main/typora_images/waitnotify/出队列02.png)

####  3.1.3 hotspot notifyAll代码(删减版)

```c++
// 这里代码简单  就是循环调用了INotify() INotify我们已经看过了 就是把waiterset的元素按顺序取出来
// 一个一个放到EntryList的头部 
// 看我下边图表示 
// The current implementation of notifyAll() transfers the waiters one-at-a-time
// from the waitset to the EntryList. This could be done more efficiently with a
// single bulk transfer but in practice it's not time-critical. Beware too,
// that in prepend-mode we invert the order of the waiters. Let's say that the
// waitset is "ABCD" and the EntryList is "XYZ". After a notifyAll() in prepend
// mode the waitset will be empty and the EntryList will be "DCBAXYZ".

void ObjectMonitor::notifyAll(TRAPS) {
  CHECK_OWNER();  // Throws IMSE if not owner.
  if (_WaitSet == NULL) {
    return;
  }

  DTRACE_MONITOR_PROBE(notifyAll, this, object(), THREAD);
  int tally = 0;
  while (_WaitSet != NULL) {
    tally++;
    INotify(THREAD);
  }

  OM_PERFDATA_OP(Notifications, inc(tally));
}
```

waiterset的连线我就少画点儿 凑活看 ：

看出什么端倪没有？ waiter的顺序 到了EntryList 变成了 倒叙 这也是为什么 我测试的时候，多个wait 在执行完notifyAll的时候 是倒着获取到锁的  ，还是那句话 JVM没有强制规定规则，所以不能以这个为依据进行业务的编写

只是大概了解一下实现原理。。 而已 

![notifyall01](https://github.com/githubforliming/blog/blob/main/typora_images/waitnotify/notifyall01.png)



```java
public class WaitNotifyTest05 {
    public static void main(String[] args) throws InterruptedException {
        Object obj = new Object();

        for (int i = 0; i < 20; i++) {
            new Thread(()->{
                synchronized (obj){
                    try {
                        obj.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }finally {
                        System.out.println(Thread.currentThread().getName()+" 获取锁");
                    }
                }
            },"线程名称"+i).start();
        }

        TimeUnit.SECONDS.sleep(2);


        new Thread(()->{
            synchronized (obj){
                try{
                    System.out.println("notifyAll  ");
                } catch (Exception e) {
                    System.out.println("异常了 ");
                } finally {
                    obj.notifyAll();
                }
            }
        }).start();
    }
}
输出结果：
notifyAll  
线程名称19 获取锁
线程名称18 获取锁
线程名称17 获取锁
线程名称16 获取锁
线程名称15 获取锁
线程名称14 获取锁
线程名称13 获取锁
线程名称8 获取锁
线程名称11 获取锁
线程名称10 获取锁
线程名称9 获取锁
线程名称12 获取锁
线程名称7 获取锁
线程名称6 获取锁
线程名称5 获取锁
线程名称4 获取锁
线程名称3 获取锁
线程名称2 获取锁
线程名称1 获取锁
线程名称0 获取锁
    
```



### 3.2  相关面试题

#### 3.2.1 sleep 与 wait的区别 

1. sleep 属于 Thread类 ， wait属于 Object类 
2. sleep 暂停当前线程指定时间，让出CPU但是不会释放锁
3. wait会释放锁 只有被调用notify/notifyAll的时候，才可能接着执行，这里一定是可能，因为他不一定能抢到锁

#### 3.2.2 wait、notify 模拟生产者和消费者  

```java
public class WaitNotifyTest06 {
     // 存放生产数据的容器
     static LinkedList<String>  list= new LinkedList<>();
     // 容器最大存放数
     static int maxCount = 1;
    public static void main(String[] args)   {
        // 生产者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果满了 wait  等待消费者消费了 通知生产者 再生产
                    if (list.size() == maxCount) {
                        try {
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    // 生产了元素 通知消费者 接着消费
                    String a = Double.valueOf(Math.random()*10000).intValue()+"";
                    list.add(a);
                    System.out.println("生产："+a);
                    list.notify();
                }
            }
        },"生产者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    if (list.size() == 0) {
                        try{
                           list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费："+list.pop());
                    list.notify();
                }
            }
        },"消费者").start();
    }
}
```

注意： 大家看到我这里判断list的大小 用的是if(lsit.size() == maxCount  ) 和 if(list.size() > = 0) 

 一个生产者和一个消费者 貌似没什么大问题  

那如果2个消费者呢 ？

```java
public class WaitNotifyTest07 {
     // 存放生产数据的容器
     static LinkedList<String>  list= new LinkedList<>();
     // 容器最大存放数
     static int maxCount = 1;
    public static void main(String[] args)   {
        // 生产者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果满了 wait  等待消费者消费了 通知生产者 再生产
                    if (list.size() == maxCount) {
                        try {
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    // 生产了元素 通知消费者 接着消费
                    String a = Double.valueOf(Math.random()*10000).intValue()+"";
                    list.add(a);
                    System.out.println("生产："+a);
                    list.notify();
                }
            }
        },"生产者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    if (list.size() == 0) {
                        try{
                           list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费01："+list.pop());
                    list.notify();
                }
            }
        },"消费者01").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    if (list.size() == 0) {
                        try{
                            list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费02："+list.pop());
                    list.notify();
                }
            }
        },"消费者02").start();
    }
}

异常喽：
Exception in thread "消费者02" java.util.NoSuchElementException
	at java.util.LinkedList.removeFirst(LinkedList.java:270)
	at java.util.LinkedList.pop(LinkedList.java:801)
	at WaitNotifyTest07.lambda$main$2(WaitNotifyTest07.java:63)
	at java.lang.Thread.run(Thread.java:748)  
```

为什么会这样呢 ？最容易想到的一条错误路径是这样的：

1. 生产者获得锁 生产 “xxoo”  
2. 消费者01 消费 “xxoo”  --> notify 
3. 这时候生产者02 获得锁，判断size() == 0  -->wait 释放锁 
4. 生产者获得锁，生产“xxxx”  --> notify
5. 消费者01 获得锁 消费 ‘xxxx’ -->notify 
6. 这是生产者02 获得锁 ，但是这时候size()==0  直接往下走 调用pop就会报错。
7. 所以 一般我们会用while 代替 if  ，获得锁之后会再判断一下wait的条件，如果条件符合再往下走 

```java
public class WaitNotifyTest07 {
     // 存放生产数据的容器
     static LinkedList<String>  list= new LinkedList<>();
     // 容器最大存放数
     static int maxCount = 1;
    public static void main(String[] args)   {
        // 生产者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果满了 wait  等待消费者消费了 通知生产者 再生产
                    if (list.size() == maxCount) {
                        try {
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    // 生产了元素 通知消费者 接着消费
                    String a = Double.valueOf(Math.random()*10000).intValue()+"";
                    list.add(a);
                    System.out.println("生产："+a);
                    list.notify();
                }
            }
        },"生产者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    while (list.size() == 0) {
                        try{
                           list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费："+list.pop());
                    list.notify();
                }
            }
        },"消费者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    while (list.size() == 0) {
                        try{
                            list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费02："+list.pop());
                    list.notify();
                }
            }
        },"消费者02").start();
    }
}
```



咦？ 又有问题了。 死锁了！！

因为一个生产者，两个消费者 需要用notifyAll 代替notify 

为什么notify会死锁  ？随便举例一种情况 

1. 生产者获得锁 生产 “xxoo” ，然后生产者又抢到锁 size() == 1 --> wait() 了 
2. 消费者01抢到锁   消费xxoo  然后自己又抢到锁  size() == 0 -- > wait()
3. 这时候消费者02 抢到锁 size() ==0 也wai()了  都wait了 

```java
public class WaitNotifyTest07 {
     // 存放生产数据的容器
     static LinkedList<String>  list= new LinkedList<>();
     // 容器最大存放数
     static int maxCount = 1;
    public static void main(String[] args)   {
        // 生产者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果满了 wait  等待消费者消费了 通知生产者 再生产
                    if (list.size() == maxCount) {
                        try {
                            list.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    // 生产了元素 通知消费者 接着消费
                    String a = Double.valueOf(Math.random()*10000).intValue()+"";
                    list.add(a);
                    System.out.println("生产："+a);
                    list.notifyAll();
                }
            }
        },"生产者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    while (list.size() == 0) {
                        try{
                           list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费："+list.pop());
                    list.notifyAll();
                }
            }
        },"消费者").start();

        // 消费者线程
        new Thread(()->{
            while (true) {
                synchronized (list){
                    // 如果没有元素 wait
                    while (list.size() == 0) {
                        try{
                            list.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    // 消费了 通知生产者接着生产
                    System.out.println("消费02："+list.pop());
                    list.notifyAll();
                }
            }
        },"消费者02").start();
    }
}
```



#### 最后附上自己公众号刚开始写 愿一起进步：



![公众号二维码](https://github.com/githubforliming/blog/blob/main/typora_images/waitnotify/公众号二维码.jpg)



**注意**： 以上文字 仅代表个人观点，仅供参考，如有问题还请指出，立即马上连滚带爬的从被窝里出来改正。
