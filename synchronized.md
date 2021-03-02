# 话说 *synchronized* 

### 一、前言

​		说起java的锁呀，我们先想到的肯定是synchronized[ˈsɪŋ krə naɪ zd]了 ，这个单词很拗口，会读这个单词在以后的面试中很加分（我面试过一些人 不会读  ，他们说的是syn开头那个单词），不会读略显不专业，不过问题不大，会用，懂原理才是最重要的。

​       内容会由简入难，有时候可以放弃一部分难的东西。 标记一下 回头再看 可能更加明朗

### 二、DEMO

​		废话不多说，先写hello world !!

​		例子： 小强 和  小明 同居了，但是只有一个厕所，他们每天必做的事情就是抢坑位，那么用代码实现要怎么写呢 ？

~~~java
/**
 * @author 木子的昼夜
 * <p>
 * 人 实体类
 */
public class Person {
    // 名字
    private String name;
    // 上厕所
    public void gotoWc() {
        Wc.useWc(this);
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}

/**
 * @author 木子的昼夜
 *
 * 厕所 实体类
 */
public class Wc {
    /**
     * 使用厕所方法
     * @param p 使用厕所的人
     */
    public static void useWc(Person p){
        try{
            System.out.println(p.getName()+" 正在使用厕所！！");
            TimeUnit.SECONDS.sleep(10);
            System.out.println(p.getName()+" 用完了！！");
        } catch (Exception e) {
            // 厕所万一坏了 也得结束使用
            System.out.println(p.getName()+" 用完了！！");
        }
    }
}


/**
 * @author 木子的昼夜
 * 这个测试 是小强与小明商量好了 说小强你先来   小强完事儿了  小明再来  
 * 这个不会发生冲突 因为是商量好的 顺序执行 
 * 大家都知道  顺序是不会出什么问题的 
 */
public class SyncTest {
    public static void main(String[] args) {
        // 小强对象
        Person xiaoqiang = new Person();
        xiaoqiang.setName("小强");

        // 小明对象
        Person xiaoming = new Person();
        xiaoming.setName("小明");

        // 上厕所
        xiaoqiang.gotoWc();
        xiaoming.gotoWc();
    }

}

~~~

上厕所过程：

![上厕所过程](https://github.com/githubforliming/blog/blob/main/typora_images/上厕所过程.png)



如果俩人没商量，自己去自己的呢 ？ 

~~~java

/**
 * @author 木子的昼夜
 */
public class SyncTest02 {
    public static void main(String[] args) {
        // 小强对象
        Person xiaoqiang = new Person();
        xiaoqiang.setName("小强");

        // 小明对象
        Person xiaoming = new Person();
        xiaoming.setName("小明");

        // 开启两个线程 谁也不理谁 自己干自己的 
        new Thread(()->xiaoqiang.gotoWc()).start();
        new Thread(()->xiaoming.gotoWc()).start();
    }

}
~~~

上厕所过程：

![上厕所过程02](https://github.com/githubforliming/blog/blob/main/typora_images/上厕所过程02.png)

上图很明显可以看出来，小强没上完呢，小明就去上了，要是小的还凑活，大的怎么办？ 画面自己想~~

这个时候大家可能想到了，厕所门上没锁吗？ 谁先进去锁住不就行了吗？

 ![bg](F:\typora_images\bg.gif) 答对了！



~~~java
/**
 * @author 木子的昼夜
 *
 * 改造后 厕所 实体类
 */
public class Wc {
    /**
     * 使用厕所方法
     * synchronized: 谁先进厕所 马上上锁 ！！
     * @param p 使用厕所的人
     */
    public static synchronized void useWc(Person p){
        try{
            System.out.println(p.getName()+" 正在使用厕所！！");
            TimeUnit.SECONDS.sleep(10);
            System.out.println(p.getName()+" 用完了！！");
        } catch (Exception e) {
            // 厕所万一坏了 也得结束使用
            System.out.println(p.getName()+" 用完了！！");
        }
    }
}

/**
 * @author 木子的昼夜
 */
public class SyncTest02 {
    public static void main(String[] args) {
        // 小强对象
        Person xiaoqiang = new Person();
        xiaoqiang.setName("小强");

        // 小明对象
        Person xiaoming = new Person();
        xiaoming.setName("小明");

        // 开启两个线程 谁也不理谁 自己干自己的  但是这次厕所有锁，
        // 谁先进去 就把锁锁住 
        new Thread(()->xiaoqiang.gotoWc()).start();
        new Thread(()->xiaoming.gotoWc()).start();
    }

}
~~~

上厕所过程：

![上厕所过程03](https://github.com/githubforliming/blog/blob/main/typora_images/上厕所过程03.png)

可以看到，小强先上完，小明再上的，这样就不会出什么问题了。

有人可能会问了，只见上锁，没有解锁，小明怎么进去的？

这就是synchronized的一个特性了，它会自动释放锁， synchronized包裹的代码执行完之后，锁就自动释放了。

所以避免了忘记释放锁，带来的尴尬~ 



### 三、 假装学术讨论

###  3.1 为什么要上锁？

  以厕所为例，自己想去吧。  提出：”共享资源“ 这个词就对了

### 3.2 对象锁  类锁

 1） *对象锁    顾名思义 就是锁一个对象* 

~~~java
/**
 * @author 木子的昼夜
 * 对象锁
 */
public class SyncObject {

    /**
     * 累加值 （共享资源）
     */
     int count = 0;
    /**
     * 锁对象
     */
    private Object lock = new Object();

    public static void main(String[] args) {
        SyncObject so = new SyncObject();

        // 线程1
        new Thread(()->{
            try {
                for (;;){
                    TimeUnit.SECONDS.sleep(2);
                    so.increaseCount();
                }
            } catch (Exception e) {
                System.err.println("错误");
            }
        },"线程1").start();

        // 线程2
        new Thread(()->{
            try {
                for (;;){
                    TimeUnit.SECONDS.sleep(2);
                    so.increaseCount();
                }
            } catch (Exception e) {
                System.err.println("错误");
            }
        },"线程2").start();
    }

    /**
     * count累加
     */
    public void increaseCount(){
        // 加锁
        synchronized (lock){
            count = count+1;
            System.out.println(Thread.currentThread().getName()+" count="+count);
        }
    }
}
~~~

![对象锁](https://github.com/githubforliming/blog/blob/main/typora_images/对象锁.png)

例子中：两个线程增加count ，锁的是lock这个对象  ，这就叫对象锁 。

有时候会看见synchronized(this) 这是什么锁 ？ this嘛 就是指当前对象，也是对象锁，

synchronized(this) 相当于  在方法上加synchronized,下边这两个方法都是锁的当前对象 

~~~java
	 /**
     * count累加
     */
    public void increaseCount(){
        // 加锁
        synchronized (this){
            count = count+1;
            System.out.println(Thread.currentThread().getName()+" count="+count);
        }
    }

    /**
     * count累加  加锁
     */
    public synchronized void  increaseCount02(){
            count = count+1;
            System.out.println(Thread.currentThread().getName()+" count="+count);
    }
~~~

​      2） *类锁  顾名思义 就是给一个类加锁* 

​     每个类load到内存之后呢，会生成一个Class类型的对象  锁的就是他 

​    其实也是锁一个对象 只是这个对象比较特殊，它代表类

~~~java
/**
 * @author 木子的昼夜
 * 对象锁
 */
public class SyncObject03 {

    /**
     * 累加值 （共享资源）
     */
    static int count = 0;
    /**
     * 锁对象
     */
    private Object lock = new Object();

    public static void main(String[] args) {
        SyncObject03 so = new SyncObject03();

        // 线程1
        new Thread(()->{
            for (;;){
                SyncObject03.increaseCount();
            }
        },"线程1").start();
        
        // 线程2
        new Thread(()->{
            for (;;){
                SyncObject03.increaseCount();
            }
        },"线程2").start();

    }

    /**
     * count累加
     */
    public synchronized static void increaseCount(){
        // 加锁
        try {
            count = count+1;
            System.out.println(Thread.currentThread().getName()+" count="+count);
            TimeUnit.SECONDS.sleep(1);
        } catch (Exception e){
            System.err.println("错误~");
        }
    }

    /**
     * count累加
     */
    public   void increaseCount02(){
        synchronized(SyncObject03.class){
            // 加锁
            try {
                count = count+1;
                System.out.println(Thread.currentThread().getName()+" count="+count);
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e){
                System.err.println("错误~");
            }
        }
    }

}
~~~

以上两种方式 都是类锁。

### 3.3 上锁方法执行的时候 可以执行当前对象未上锁方法吗？

   这是一个用脚指头就能想到的答案，但是好多面试官问。 问了之后呢 你就蒙了~~ 难道不能？ 

  答案是：能!   

  为什么能呢？因为爱所以爱~~   错了，重来   ..   因为能所以能~~ 

  小明在吃饭，给碗上个锁，别人不能用， 那小明能同时看他的偶像邓紫棋唱歌吗 ？ 

 谁要是说不可以，以后吃饭不让他玩手机、pad、电脑 。 就让他吃吃吃 （那是猪）

![hx](https://github.com/githubforliming/blog/blob/main/typora_images/hx.gif)

~~~java
/**
 * @author 木子的昼夜
 *  可能出现的面试题 
 */
public class SyncObject04 {


    private Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        SyncObject04 so = new SyncObject04();

        // 线程1
        new Thread(()->{
            so.increaseCount();
        },"线程1").start();

        Thread.sleep(2000);

        // 线程2
        new Thread(()->{
            so.lookMv();
        },"线程2").start();
    }

    /**
     * 吃饭
     */
    public void increaseCount(){
        // 加锁
        synchronized (this){
            try{
                System.out.println("吃饭 ");
                // 也可以在吃饭这里 跟妹子聊天 
                chatWithGirl();
                TimeUnit.SECONDS.sleep(10);
            } catch (Exception e) {
                System.err.println("饭掉地上了~");
            }
            System.out.println("吃完饭 ");
        }
    }

    /**
     * 看演唱会视频
     */
    public  void  lookMv(){
        System.out.println("看演唱会视频");
    }
    
    /**
     * 看跟美女聊天
     */
    public  void  chatWithGirl(){
        System.out.println("跟美女聊天");
    }
}
~~~



![nolock](https://github.com/githubforliming/blog/blob/main/typora_images/nolock.png)



### 3.4 可重入

  能一句话总结吗？ 

 咳咳： 同一线程可以调用加了同一把锁的两个方法 不会阻塞。

例子：同一个人可以用同一双筷子（筷子加锁），吃不同的菜~~

吃鱼的时候获取了锁，在吃鱼方法里调用吃沙拉方法，是可以调用成功了 因为两个方法用的同一把锁

~~~java
/**
 * @author 木子的昼夜
 */
public class TestC {
    /**
     * 一双筷子 只能一个人同一时间使用（一个线程）
     */
    Object chopsticks   = new Object();

    public static void main(String[] args) {
        TestC c = new TestC();
        new Thread(()->{
            c.eatFish();
        }).start();

    }

    /**
     * 吃
     */
    public void eatFish( ) {
        synchronized (chopsticks) {
           try {
              System.out.println("吃 鱼");
              Thread.sleep(2000);
              eatSalad();
           } catch (Exception e){ }
        }
    }

    /**
     * 吃沙拉
     */
    public void eatSalad() {
        synchronized (chopsticks) {
            try {
                System.out.println("吃 沙拉");
                Thread.sleep(2000);
                eatFish();
            } catch (Exception e){ }
        }
    }
}

~~~



### 3.5 底层实现

####    (1) *简单版* 

​	jdk1.6之前 synchronized 是 重量级锁  什么是重量级锁? 就是每次锁都会去找操作系统申请锁。

​    jdk1.6及以后改进为锁升级 

​    简单思路是： 

​     synchronized（object）

1. 线程A 第一个访问 
2. **偏向锁**  只在object的markword 中记录线程A的线程ID
3. 如果线程A 又进来访问 一看markword的线程号是自己 那就直接用 
4. 这时候线程B 来了 ，线程B 一看我擦？ 有人占用了锁！ 
5. 线程B 会循环死等  ，类似在厕所门口，敲敲门问问线程A 你好了吗？敲敲门问问线程A 你好了吗？敲敲门问问线程A 你好了吗？敲敲门问问线程A 你好了吗？
6. 线程B的这一操作，用术语将叫： **自旋锁**
7. 线程B 问10次之后，得不到锁，就会升级为**重量级锁** （去操作系统申请资源）
8. 无锁->偏向锁->自旋锁->重量级锁
9.  锁 一般 。。 只升级 不降级  

#### （2)复杂版

1. ##### CAS 简单叙述 了解入门

  ![cas](https://github.com/githubforliming/blog/blob/main/typora_images/cas.png)

 

   什么是ABA问题，假如你有媳妇儿，我说假如~  以偷零花钱为例

   https://www.processon.com/view/link/603c96ca07912913b4f2c55f

  ![偷钱](https://github.com/githubforliming/blog/blob/main/typora_images/偷钱.png)

​    这是一个故事： 小强偷零花钱请小明吃饭的故事 

   ABA 就是：

 小强媳妇儿 出门看的钱是10万 （A）  

 小强偷拿1万 剩余9万（B）    

小强找小月借了1万 放回去 总共10万（A） 

小强媳妇儿回来 一看是10万，很满意。 但是 她不知道  这是偷梁换柱啊 



后来小强媳妇儿看了我的博客 ， 发现了秘密 ，她应该怎么解决呢 ？

关键字：version 版本号 

她上班之前，在家里的存款上用笔写了一个版本，小强如果再偷钱，再还回来，这个版本就变了（+1）

这样就解决了ABA问题 

##### 2. 上锁过程  

 ![上锁过程](https://github.com/githubforliming/blog/blob/main/typora_images/上锁过程.png)

 重度竞争： 耗时过长 自选过多  wait等 

新建对象可能直接是匿名偏向 ( 如果默认开启了偏向锁) ，因为没有偏向任何一个线程，所以是匿名偏向

JVM默认不开启 延迟4秒后才会开启 偏向锁 

##### 3. *new 一个对象 长什么样 ？*  markword 是啥 

![对象样子](https://github.com/githubforliming/blog/blob/main/typora_images/对象样子.png)



  **1. 查看工具 ： JOL (Java Object Layout )**

   直接maven引入就可以使用了

 ~~~xml
 <dependencies>
     <dependency>
         <groupId>org.openjdk.jol</groupId>
         <artifactId>jol-core</artifactId>
         <version>0.9</version>
     </dependency>
</dependencies>
 ~~~

~~~java
/**
 * @author 木子的昼夜
 */
public class Person {
    long  money;

    public long getMoney() {
        return money;
    }

    public void setMoney(long money) {
        this.money = money;
    }
}

/**
 * @author 木子的昼夜
 */
public class JolTest {
    public static void main(String[] args) {
        Person p = new Person();
        System.out.println(ClassLayout.parseInstance(p).toPrintable());
    }
}

// 输出结果 
Person object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 
      4     4        (object header)                           00 00 00 00 
      8     4        (object header)                           43 c1 00 f8 
     12     4        (alignment/padding gap)                  
     16     8   long Person.money                              0
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total

~~~



![jol01.png](https://github.com/githubforliming/blog/blob/main/typora_images/jol01.png)



咦？ 不是要写synchronized 吗 ？  

**2. markword** 

我给对象p 加锁，然后输出 layout 可以发现 markword 改变了 

所以呢~~ 锁信息 是记录再markword中的  

~~~java
/**
 * @author 木子的昼夜
 */
public class JolTest {
    public static void main(String[] args) {
        Person p = new Person();
        System.out.println(ClassLayout.parseInstance(p).toPrintable());

        synchronized (p) {
            System.out.println(ClassLayout.parseInstance(p).toPrintable());
        }
    }
}
~~~

![mk01](https://github.com/githubforliming/blog/blob/main/typora_images/mk01.png)







markword 这么厉害吗? 不 ！ 它还能更厉害 。 我们看一下 它里边都记录了一些什么信息 

先暂时看最后3bit 其他 不是很了解   这个时候可以对比上边layout的输出 看一下

刚开始是01  加锁之后变成了00  （没有开启偏向锁 直接到轻量级锁）

![mk02](https://github.com/githubforliming/blog/blob/main/typora_images/mk02.png)

![mk03](https://github.com/githubforliming/blog/blob/main/typora_images/mk03.png)



先看最后2bit:

---



**00**:轻量级锁  自旋锁 

​       自旋锁，耗CPU资源   是在用户态操作 不关联内核态 

​		两个线程 争着把自己的Lock Record 放到markword中 

​        谁先放进去，谁先获得锁，另一个人接着cas 去放 

​       ![自旋锁](https://github.com/githubforliming/blog/blob/main/typora_images/自旋锁.png)

​        Lock Record 指向的是什么呢  是无锁状态的markword 

​      ![mk0](https://github.com/githubforliming/blog/blob/main/typora_images/mk04.png)

 这就解释了 为什么hashcode不丢失的问题  因为有备份记录 

​     

​    这里锁重入： 上边提到了，锁重入 ，锁每进一次，都会加一个LR  从第二个LR开始 指向的就是一个null 

​    等锁退出 也就是monitorexit（锁代码块执行完 或 抛异常）的时候LR -1 ,LR -1 ,LR -1 一直减  ，退一次减一次 

---



**10:重量级锁**

​      向OS 申请锁，进了内核态  ， c++ 新建了一个*object monitor*对象  markword中放的就是这个 指针 （java中就是个地址或者是ID ）

​     重量级锁，都在一个队列里等着，比较不消耗CPU资源 

​    可重入锁： 重量级是记录再object moniter 的某个属性上 



​    什么时候自选上升为重量级锁： 

	1. 自选次数超过10次  或者 自选的线程数超过CPU的一半  
 	2. jdk1.6之前 -XX:PreBlockSpin可以调整 自选超过多少次升级 
 	3. jdk1.6之后加入了自适应 Adapative Self Sping  JVM自己个儿控制 

---



**11**:GC回收标记

---



**01**: 再看倒数第三位  

​		 **0**：无锁   

​		**1**：偏向锁      放线程ID ，  c++实现是用的指针

##### 

**3. synchronized 编译成字节码  会有两个单词  monitorenter   monitorexit**  

什么时候monitorexit呢 ， 代码执行完 ，或者是异常发生   这就是synchronized 自动释放锁的原理

##### 4. 为什么有自旋锁还需要重量级锁

(1) 自旋是消耗CPU资源的，如果锁的时间长，或者自旋线程多，CPU会被大量消耗

(2) 重量级锁有等待队列，所有拿不到锁的进入等待队列，不需要消耗CPU资源

##### 5. 偏向锁是否一定比自旋锁效率高？

(1)不一定 当你知道肯定存在多线程竞争的时候，偏向锁会涉及锁撤销，这时候自旋锁会比较好一点

(2) JVM启动过程就会很很多线程竞争，所以默认不开启偏向锁，过一段时间才会开启

(3) -XX:*BiasedLockingStartupDelay* = 0   默认是 4 秒 













最后附上自己的微信公众号 刚开始做  愿一起进步： 
 ![公众号图片](https://github.com/githubforliming/blog/blob/main/typora_images/公众号二维码.jpg)
 

**注意**： 以上文字 仅代表个人观点，仅供参考，如有问题还请指出，立即马上连滚带爬的从被窝里出来改正。

