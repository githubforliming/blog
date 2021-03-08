# 话说 CAS 

# 一、前言

1. cas 一般认为是compare and swap 也可以认为是compare and set 

2. cas涉及三个值

3. （1） P  变量内存地址 

​        （2）E  期望值 ,CPU做计算之前拿出来的旧值

​          (3)   X 需要设置的新值 

   原子操作为： 拿出内存地址当前的值A ,比较A == E ? 是 ： 设置P内存的值为X    否：结束。。失败

4.  
    (1) 第一篇 话说synchronized  画过CAS的流程图   咱们再来一张？

![cas](https://github.com/githubforliming/blog/blob/main/typora_images/cas/cas.png)



   (2)  CAS面试经常问的一个是ABA 问题 什么是ABA ? 上图 

![cas](https://github.com/githubforliming/blog/blob/main/typora_images/cas/aba.png)

 (3) 有人说ABA 不影响啊  我反正期望的值是A  你最后是A就得了呗   

​     这个还要看具体的业务，拿生活中例子来说，银行职员小孙，偷拿了银行100万，

   然后去投资赚了20万，最后把100万还回去。 你细品。。 银行能允许吗

（4）ABA 的解决方案  版本 version  怎么解决 ？ 

![cas](https://github.com/githubforliming/blog/blob/main/typora_images/cas/abav.png)

# 二、DEMO

### 1. CAS  简单使用  

假如有一个值 int  count  ，2个线程  每个线程给count加5000次 1 
按道理说 每个人给你5000 你应该有1万块  

```java
public class CasTest {
    public  static int count = 0;
    public static void main(String[] args) throws InterruptedException {
        new Sub("第一个").start();
        new Sub("第二个").start();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("count="+CasTest.count);
    }
}

class Sub extends  Thread{
    private String name;
    public Sub(String name) {
        this.name = name;
    }
    public void run() {
        System.out.println(name+"开始+");
        for (int i = 0; i < 5000; i++) {
            try {
                CasTest.count = CasTest.count+1;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        System.out.println(name+"加完了..");
    }
}

执行结果：
第一个开始+
第二个开始+
第一个加完了..
第二个加完了..
count=6811
```

**解决方案1** 加锁synchronized 或者 lock 都可以 

```java
public class CasTest02 {
    public  static Integer count = 0;
    public static void main(String[] args) throws InterruptedException {
        new Sub02("第一个").start();
        new Sub02("第二个").start();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("count="+CasTest02.count);
    }
}

class Sub02 extends  Thread{
    private String name;
    public Sub02(String name) {
        this.name = name;
    }
    public void run() {
        System.out.println(name+"开始+");
        for (int i = 0; i < 500; i++) {
            try {
                // 加锁
                synchronized (CasTest02.class) {
                    CasTest02.count = CasTest02.count+1;
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        System.out.println(name+"加完了..");
    }
}
```

​      **解决方案2：** CAS  java自带的原子类 AtomicInteger 

​     读者读到这里可以了解一下LongAdder

```java
public class CasTest03 {
    public  static AtomicInteger count = new AtomicInteger(0);
    public static void main(String[] args) throws InterruptedException {
        new Sub03("第一个").start();
        new Sub03("第二个").start();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("count="+CasTest03.count);
    }
}

class Sub03 extends  Thread{
    private String name;
    public Sub03(String name) {
        this.name = name;
    }
    public void run() {
        System.out.println(name+"开始+");
        for (int i = 0; i < 5000; i++) {
            // 加锁
            CasTest03.count.incrementAndGet();
        }
        System.out.println(name+"加完了..");
    }
}
执行结果：
第一个开始+
第二个开始+
第二个加完了..
第一个加完了..
count=10000
```

### 2. ABA 问题  

这里简单复现一个ABA问题  可能不是很精确  读者朋友体会意思即可 

```java
public class CasTest04 {
    public  static AtomicInteger count = new AtomicInteger(0);
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{

            count.compareAndSet(0,1);
            count.compareAndSet(1,0);
            System.out.println("线程1 把count从0 修改为1  再从1  修改为0  ");
        },"线程1").start();

        new Thread(()->{
           try {
               TimeUnit.SECONDS.sleep(1);
               // 这里是0  但是已经不是他所希望的那个0 了
               count.compareAndSet(0,4);
               System.out.println("线程2 把count 从0 修改为4");
           } catch (Exception e) {
               e.printStackTrace();
           }
        },"线程2").start();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("count="+count);
    }
}
执行结果：
线程1 把count从0 修改为1  再从1  修改为0  
线程2 把count 从0 修改为4
count=4
```

**ABA解决** 加版本

```java
public class CasTest05 {
    public  static AtomicStampedReference<Integer> count = new AtomicStampedReference<>(0,0);
    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            try {
                // 等1秒  让线程2拿到版本
                TimeUnit.SECONDS.sleep(1);
                boolean res = count.compareAndSet(
                    0, 
                    1, 
                    count.getStamp(), 
                    count.getStamp() + 1);
                boolean res2 = count.compareAndSet
                    (1, 
                     0, 
                     count.getStamp(), 
                     count.getStamp() + 1);
                System.out.println("线程1 把count从0 修改为1  再从1  修改为0  "
                                   + ( res2 ? "成功！":"失败！"));
            }catch (Exception r){
                r.printStackTrace();
            }

        },"线程1").start();

        new Thread(()->{
           try {
               // 版本 
               int stamp = count.getStamp();
               TimeUnit.SECONDS.sleep(2);
               // 这里是0  但是已经不是他所希望的那个0 了  版本已经变了 
               boolean res = count.compareAndSet(0,4,stamp,stamp+1);
               System.out.println("线程2 把count 从0 修改为4" 
                                  + ( res ? "  成功！":"  失败！"));
           } catch (Exception e) {
               e.printStackTrace();
           }
        },"线程2").start();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("count="+count.getReference());
    }
}
```



 

# 三、 假装学术讨论

```java
/**
 * @author 木子的昼夜
 */
public class CasTest {
    public static void main(String[] args) {
        // JUC包里的原子类  线程安全的 
        AtomicInteger ai = new AtomicInteger();
        // 加1 并返回 
        Integer res = ai.incrementAndGet();
        System.out.println(res);
        
        // 上边这句话的意思相当于 
        int i = 0;
        i = i+1;
        int resi = i;
        System.out.println(resi);
    }
}

 /**
     * Atomically increments by one the current value.
     * 当前值自动加1 
     * @return the updated value 
     * 返回更新了的值
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
	}

	// Unsafe.java#getAndAddInt
    public final int getAndAddInt(Object obj, long offset, int step) {
        int E;
        do {
            var5 = this.getIntVolatile(obj, offset);
            // 这里不一定能一次就成功哦  会执行多次 
            // 跟一个姑娘表白 一次不成 你就灰溜溜走了 ？ 活该单身..
        } while(!this.compareAndSwapInt(obj, offset, E, E + step));

        return var5;
    }
	// Unsafe.java#getIntVolatile 这个方法是获 对象obj 内存开始地址 相对偏移位置offset 对应的值
    // 说白了 就是获取对象对应字段的值 
	public native int getIntVolatile(Object obj, long offset);

    // 
	public final native boolean compareAndSwapInt(
        Object obj,  // 被修改属性的对象 
        long offset, // 被修改字段相对于当前对象内存首地址偏移量 可以通过他直接去内存拿数据
        int E, // 期望值 
        int X);// 需要设置的新值 
```

```c++
// obj设置属性的对象 offset 字段相对于类内存起始位置偏移量  e期望值  x要设置的值 
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  // 转换对象格式 jobject- > oop 
  oop p = JNIHandles::resolve(obj);
  if (p == NULL) {
     // 获取filed对应的内存地址 对象起始地址+偏移量
    volatile jint* addr = (volatile jint*)index_oop_from_field_offset_long(p, offset);
    // 把addr内存对应的值 设置为x 前提是内存值要等于e 
    return RawAccess<>::atomic_cmpxchg(addr, e, x) == e;
  } else {
    // 
    assert_field_offset_sane(p, offset);
    return HeapAccess<>::atomic_cmpxchg_at(p, (ptrdiff_t)offset, e, x) == e;
  }
} UNSAFE_END
// 这里就调用了 atomic_cmpxchg： 系统方法 ： 原子比较并交换计数值。
atomic_cmpxchg(void* addr, T compare_value, T new_value) {
	if (is_hardwired_primitive<decorators>()) {
        const DecoratorSet expanded_decorators = decorators | AS_RAW;
        return PreRuntimeDispatch::atomic_cmpxchg<expanded_decorators>
            (addr, compare_value, new_value);
    } else {
        return RuntimeDispatch<decorators, T, BARRIER_ATOMIC_CMPXCHG>::atomic_cmpxchg
            (addr, compare_value, new_value);
      }
}


```

再底层就是系统级别的实现了，CPU 实现Atomic ，我也是看文章看得，说是有2中方式

1. 使用总线锁   总线就是老大  CPU小c 给总线发一个LOCK信号  总线收到之后  小c就独占共享内存了，其他CPU 
   就没有使用权限了 ，数据夸缓存行时使用总线锁 这时候不能用缓存锁
2. 使用缓存锁  大多数时候 我们只需要保证对某一块内存的操作时原子性即可，缓存锁就是如果内存区域被缓存再处理器的缓存行中，并且操作的时候缓存行被锁定了，那么当处理器计算完回写到内存时，处理器就把缓存行的地址修改了，如果这时有两一个处理器回写数据到缓存行，咦？ 失效了。。 





下边连接是我准备写文章的目录，如果有读者觉得需要添加，请微信公众直接发消息给我。

https://www.processon.com/view/link/5f57817963768959e2dc7dca

#### 最后附上自己公众号刚开始写 愿一起进步：



![公众号二维码](https://github.com/githubforliming/blog/blob/main/typora_images/cas/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)



**注意**： 以上文字 仅代表个人观点，仅供参考，如有问题还请指出，立即马上连滚带爬的从被窝里出来改正。
