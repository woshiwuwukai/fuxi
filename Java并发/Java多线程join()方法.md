# 一、join()方法的作用

当前线程中的其它子线程调用`join()`方法时，当前线程会进入等待状态，一直到调用`join()`方法的线程执行结束为止。

通常用于在main()主线程内，等待其它线程完成再执行main()主线程。



# 二、join()方法的用法示例

```java
public class Demo {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t1线程,i" + i);
                }

            }
        });

        Thread thread2 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t2线程,i" + i);
                }

            }
        });

        Thread thread3 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t3线程,i" + i);
                }

            }
        });

        thread.start();
        thread2.start();
        thread3.start();

        thread.join();
        thread2.join();
        thread3.join();
        System.out.println("main线程执行完毕");

    }
}

```

如果不使用`join()`方法的话，子线程和主线程是没关系的，互不相干，各走各的。下面为不使用`join()`方法时的结果:

```java
main线程执行完毕
我是t2线程,i0
我是t3线程,i0
我是t1线程,i0
我是t3线程,i1
我是t2线程,i1
我是t3线程,i2
我是t1线程,i1
我是t3线程,i3
我是t2线程,i2
我是t3线程,i4
我是t1线程,i2
我是t1线程,i3
我是t1线程,i4
我是t2线程,i3
我是t2线程,i4
```

可以看到，main线程这时最先执行完了，并不会等三个子线程执行完毕。

如果使用`join()`方法的话，则会等到使用了`join()`方法的线程执行完，才会继续执行main线程。下面为使用`join()`方法时的结果:

```Java
我是t2线程,i0
我是t1线程,i0
我是t1线程,i1
我是t1线程,i2
我是t1线程,i3
我是t1线程,i4
我是t3线程,i0
我是t3线程,i1
我是t3线程,i2
我是t3线程,i3
我是t3线程,i4
我是t2线程,i1
我是t2线程,i2
我是t2线程,i3
我是t2线程,i4
main线程执行完毕
```

可以看到，main线程最后才执行完毕。而且这时这三个子线程间也是毫不相干的，各走各的，只有主线程会被挂起。

那么，如果有个子线程没用`join()`方法呢？

```Java
public class Demo1 {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t1线程,i" + i);
                }

            }
        });

        Thread thread2 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t2线程,i" + i);
                }

            }
        });

        Thread thread3 = new Thread(new Runnable() {

            @Override
            public void run() {
                for (int i = 0; i < 5; i++) {
                    System.out.println("我是t3线程,i" + i);
                }

            }
        });

        thread.start();
        thread2.start();
        thread3.start();

        thread.join();
        // thread2.join();
        thread3.join();
        System.out.println("main线程执行完毕");

    }
}

```

结果如下：

```Java
我是t1线程,i0
我是t3线程,i0
我是t2线程,i0
我是t3线程,i1
我是t1线程,i1
我是t3线程,i2
我是t2线程,i1
我是t3线程,i3
我是t1线程,i2
我是t3线程,i4
我是t2线程,i2
我是t1线程,i3
我是t2线程,i3
我是t1线程,i4
main线程执行完毕
我是t2线程,i4
```

可以看到，main线程并不一定是最后执行的，这是因为t2线程没有用`join()`方法,主线程也就并不会等待t2线程执行完再执行了。

那么，如果这是我想控制三个子线程的执行顺序呢？

例如，现在有T1、T2、T3三个线程，你怎样保证T2在T1执行完后执行，T3在T2执行完后执行？

```Java
public class Demo2 {
 
	public static void main(String[] args) {
		Thread thread = new Thread(new Runnable() {
 
			@Override
			public void run() {
				for (int i = 0; i < 5; i++) {
					System.out.println("我是t1线程,i" + i);
				}
 
			}
		});
 
		Thread thread2 = new Thread(new Runnable() {
 
			@Override
			public void run() {
				try {
					thread.join(); #1
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				for (int i = 0; i < 5; i++) {
					System.out.println("我是t2线程,i" + i);
				}
 
			}
		});
 
		Thread thread3 = new Thread(new Runnable() {
 
			@Override
			public void run() {
				try {
					thread2.join(); #2
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				for (int i = 0; i < 5; i++) {
					System.out.println("我是t3线程,i" + i);
				}
 
			}
		});
        
		thread.start();
		thread2.start();
		thread3.start();
	}
 
}
```

结果如下：

```Java
我是t1线程,i0
我是t1线程,i1
我是t1线程,i2
我是t1线程,i3
我是t1线程,i4
我是t2线程,i0
我是t2线程,i1
我是t2线程,i2
我是t2线程,i3
我是t2线程,i4
我是t3线程,i0
我是t3线程,i1
我是t3线程,i2
我是t3线程,i3
我是t3线程,i4
```

在#1处，t2线程中，t1调用`join()`方法，此时t2会等t1执行完，再执行。

在#1处，t3线程中，t2调用`join()`方法，此时t3会等t2执行完，再执行。

这样，就实现了上面的需求了。



# 三、join()方法的原理

```Java
public final void join() throws InterruptedException {
    //join()等同于join(0)
    join(0);            
}
 
//注意被同步修饰符
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;
 
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
 
    if (millis == 0) {
        while (isAlive()) {// 循环+isAlive()方法就是用来判断当前线程是否存活
            wait(0);  //join(0)等同于wait(0)，即wait无限时间直到被notify
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);
            now = System.currentTimeMillis() - base;
        }
    }
}
```

- join()方法的底层是利用`wait()`方法实现。
- join()方法是一个同步方法，当主线程调用t1.join()方法时，主线程先获得了t1对象的锁。
- join()方法中主线程调用了t1对象的wait()方法，使主线程进入了t1对象的等待池。

`wait():调用该方法的线程进入WAITING状态，只有等待另外线程的通知或被中断才会返回，需要注意，调用wait()方法后，会释放对象的锁。`

### join()方法的两种等效写法

源码中：

```
 while (isAlive()) {
            wait(0);
        }
```

这就是为什么线程会一直等待，因为它调用的就是`wait()`方法呀。

### join()方法的第二种等效写法

所以join()方法又等同于以下代码：

```
synchronized(thread1)){
    thread1.wait();
}
```

# 四、join()方法源码疑问

​        join源码中，只看到join()方法调用了wait()方法，但是没看到有调用notify() 或者 notifyAll() 系列的唤醒方法，那它是怎么唤醒的呢？如果不唤醒，那不就一直等待下去了吗？

原来啊，在java中，Thread类线程执行完run()方法后，一定会自动执行notifyAll()方法。因为线程在死亡的时候会释放持用的资源和锁，自动调用自身的notifyAll方法。

这是notifyAll()非常重要的隐藏知识点，这个细节隐藏在Java的Native方法中，所以一般不会被人发现。我们观察C/C++源码，如下：

```c++
oid JavaThread::exit(booldestory_vm, ExitTypeexit_type);
static void ensure_join(JavaThread*thread) {
	Handle threadObj(thread, thread -> threadObj());
	ObjectLocker lock(threadObj, thread);
	thread -> clear_pending_exception();
	java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
	java_lang_Thread::set_thread(threadObj(), NULL);
     //下行执行了notifyAll()操作
	lock.notify_all(thread);
	thread -> clear_pending_exception();
}
```


其中ensure_join就是执行join()方法，等方法执行结束时，此行代码lock.notify_all(thread);意思是通过notifyAll()唤醒了所有等待线程。

所以在使用线程的时候，要特别注意 。

# 六、join()方法的死锁

java的join方法中，这里有一个坑，就是下面这个方法

```Java
Thread.currentThread().join();
```

我们都知道 ，join方法的作用是阻塞线程，即等待线程结束，才继续执行。

如果调用了线程自身调用join()这个方法，那么线程一直在阻塞，无法终止。

因为它自己在等待自己结束；这无疑会造成死锁!