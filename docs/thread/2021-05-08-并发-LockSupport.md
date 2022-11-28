

## LockSupport

LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。

LockSupport中的park()和 unpark()的作用分别是阻塞线程和解除阻塞线程。

总之，比wait/notify，await/signal更强。

3种让线程等待和唤醒的方法

- 使用Object中的wait()方法让线程等待，使用object中的notify()方法唤醒线程
- 使用JUC包中Condition的await()方法让线程等待，使用signal()方法唤醒线程
- LockSupport类可以阻塞当前线程以及唤醒指定被阻塞的线程

## waitNotify限制

Object类中的wait和notify方法实现线程等待和唤醒

```
public class WaitDemo {

    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        new Thread(()->{
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName()+" come in.");
                try {
                    lock.wait();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName()+" 换醒.");
        }, "Thread A").start();

        Thread.sleep(2000);

        new Thread(()->{
            synchronized (lock) {
                lock.notify();

                System.out.println(Thread.currentThread().getName()+" 通知.");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "Thread B").start();
    }
}
```

wait和notify方法必须要在同步块或者方法里面且成对出现使用，否则会抛出java.lang.IllegalMonitorStateException。

调用顺序要先wait后notify才OK。

## awaitSignal限制

Condition接口中的await后signal方法实现线程的等待和唤醒，与Object类中的wait和notify方法实现线程等待和唤醒类似。

```
public class ConditionAwaitSignalDemo {
    public static void main(String[] args) {

        ReentrantLock lock = new ReentrantLock();
        Condition condition = lock.newCondition();

        new Thread(()->{

            try {
                System.out.println(Thread.currentThread().getName()+" come in.");
                lock.lock();
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }

            System.out.println(Thread.currentThread().getName()+" 换醒.");
        },"Thread A").start();

        new Thread(()->{
            try {
                lock.lock();
                condition.signal();
                System.out.println(Thread.currentThread().getName()+" 通知.");
            }finally {
                lock.unlock();
            }
        },"Thread B").start();
    }

} 
```

await和signal方法必须要在同步块或者方法里面且成对出现使用，否则会抛出java.lang.IllegalMonitorStateException。

调用顺序要先await后signal才OK。

## LockSupport方法介绍

传统的synchronized和Lock实现等待唤醒通知的约束

- 线程先要获得并持有锁，必须在锁块(synchronized或lock)中
- 必须要先等待后唤醒，线程才能够被唤醒

LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。

LockSupport类使用了一种名为Permit（许可）的概念来做到阻塞和唤醒线程的功能，每个线程都有一个许可（permit），permit只有两个值1和零，默认是零。

可以把许可看成是一种(0.1)信号量（Semaphore），但与Semaphore不同的是，许可的累加上限是1。

通过park()和unpark(thread)方法来实现阻塞和唤醒线程的操作

park()/park(Object blocker) - 阻塞当前线程阻塞传入的具体线程

```
public class LockSupport {

    ...
    
    public static void park() {
        UNSAFE.park(false, 0L);
    }

    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
    
    ...
    
}

```

permit默认是0，所以一开始调用park()方法，当前线程就会阻塞，直到别的线程将当前线程的permit设置为1时，park方法会被唤醒，然后会将permit再次设置为0并返回。

unpark(Thread thread) - 唤醒处于阻塞状态的指定线程

```
public class LockSupport {
 
    ...
    
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
    
    ...

}
```

调用unpark(thread)方法后，就会将thread线程的许可permit设置成1（注意多次调用unpark方法，不会累加，pemit值还是1）会自动唤醒thead线程，即之前阻塞中的LockSupport.park()方法会立即返回。

## LockSupport案例解析

```
public class LockSupportDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread a = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "come in");
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + "unpark");
        }, "a");

        a.start();

        Thread.sleep(3000);

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "come in");
            LockSupport.unpark(a);

        }, "b").start();

    }
}
```

LockSupport是用来创建锁和共他同步类的基本线程阻塞原语。

LockSuport是一个线程阻塞工具类，所有的方法都是静态方法，可以让线程在任意位置阻塞，阻寨之后也有对应的唤醒方法。归根结底，LockSupport调用的Unsafe中的native代码。

LockSupport提供park()和unpark()方法实现阻塞线程和解除线程阻塞的过程

LockSupport和每个使用它的线程都有一个许可(permit)关联。permit相当于1，0的开关，默认是0，调用一次unpark就加1变成1，调用一次park会消费permit，也就是将1变成0，同时park立即返回。

如再次调用park会变成阻塞(因为permit为零了会阻塞在这里，一直到permit变为1)，这时调用unpark会把permit置为1。每个线程都有一个相关的permit, permit最多只有一个，重复调用unpark也不会积累凭证。

形象的理解: 线程阻塞需要消耗凭证(permit)，这个凭证最多只有1个。当调用park方法时,如果有凭证，则会直接消耗掉这个凭证然后正常退出。如果无凭证，就必须阻塞等待凭证可用。而unpark则相反，它会增加一个凭证，但凭证最多只能有1个，累加无放。

### 为什么可以先唤醒线程后阻塞线程？

因为unpark获得了一个凭证，之后再调用park方法，就可以名正言顺的凭证消费，故不会阻塞。

### 为什么唤醒两次后阻塞两次，但最终结果还会阻塞线程？

因为凭证的数量最多为1（不能累加），连续调用两次 unpark和调用一次 unpark效果一样，只会增加一个凭证；而调用两次park却需要消费两个凭证，证不够，不能放行。