---
layout: post
title: "jvm中的synchronized"
description: ""
category: "j2ee"
tags: []
---
背景：   
　　Java的优势在于从语言层面就提供了对多线程同步的支持，而不像C++，要通过其它库才支持多线程。synchronized是JDK1.5前唯一实现同步的方式，即使1.5之后有了J.U.C包，但synchronized使用还是非常的广泛。   
　　在使用synchronized时，我们并不知道JVM是如何保证资源同步，可以说我们对synchronized不是很了解，有些疑问常常有脑海里浮现，比如，synchronized同步时，线程是否有先后顺序，如果有，是FIFO还是LIFO呢?还有性能怎么样，当然，性能有很多测量的工具。为此，我想深入了解一下synchronized，查找过的一些资料我会在后面附录里贴出来，还有自己写的测试代码。   
　　synchronized使用内置锁（也叫对象锁或对象监视器Monitor）实现同步，任何不等于NULL的对象都可以作为“锁”，当作用在方法上时，锁住的便是对象实例（this）；作用在静态方法时锁住的便是对象对应的class；当作用于代码块时，锁住的便是synchronized语法括号内的对象。   

原理：   
　　进入synchronized区域（字节码由monitorenter和monitorexit表示）的线程会争夺锁资源。   
![线程抢占锁原理图](/assets/img/synchronized.png "线程抢占锁原理图")   
　　JVM使用\_cxq、\_entryList（\_entrySet）、\_waitSet队列来实现线程状态的转换、资源抢占，最终达到同步的目的。   
　　线程Thread-x首先通过spin（自旋），利用CAS尝试获取锁，这里可以看出，锁竞争是“非公平”的，因为先到达的线程有可能在\_cxq排队，而后到达的Thread-x线程已经开始参与锁竞争，如果得到锁，Thread-x将获取执行代码的权利，如果失败，Thread-x将通过CAS把自己放入\_cxq队列，并将线程状态更新为阻塞(park)，\_cxq是一个LIFO（后进先出）的队列，指针永远指向head。\_entryList也是一个线程等待队列，当\_owner(持有当前锁的线程)释放锁时，如果发现\_cxq != null && \_entryList == null，那么，\_owner将\_cxq中的线程节点移动到\_entryList，同时\_owner会唤醒（unpark）它的下一个线程节点参与锁竞争，这里是参与，不是把锁直接给下一个线程节点。   
　　那些调用wait方法被阻塞的线程被放置到\_waitSet队列。   
　　JVM底层实现synchronized是一个复杂的过程，远不像我说的这样简单，锁资源的体现在objectMonitor.cpp中，附录有源码。   

验证：   
　　1.既然\_cxq是一个LIFO队列，那么，进入排队的线程应该也是按LIFO的顺序执行synchronized代码块，这里的进入排队的线程是指进入\_cxq队列的线程，如果没有这个前提条件，线程竞争锁应该是无序的。下面是测试代码：  
 
```java   
import java.util.concurrent.locks.*;
import java.util.concurrent.TimeUnit;

public class CountDownTest {
	private int i;
	private final ReentrantLock lock = new ReentrantLock();

	public CountDownTest(int i) {
		this.i = i;
	}

	public void synchronizedDown() {
		synchronized(this){
		
			try {
				TimeUnit.SECONDS.sleep(2L);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		
			System.out.println("enter : " + Thread.currentThread().getName());
			--i;
			
		}	
	}
	
	public void lockDown() {
		lock.lock();
			try {
				TimeUnit.SECONDS.sleep(2L);
				System.out.println("enter : " + Thread.currentThread().getName());
				--i;
			} catch (InterruptedException e) {
				e.printStackTrace();
			}finally{
				lock.unlock();
			}
	}
}

public class Test implements Runnable {

	private CountDownTest countDown;

	public Test(CountDownTest countDown) {
		super();
		this.countDown = countDown;
	}

	public void run() {
		System.out.println("ready : " + Thread.currentThread().getName());
		countDown.synchronizedDown();
		//countDown.lockDown();
	}

	public static void main(String[] args) {
		int i = 8;
		CountDownTest countDown = new CountDownTest(i);
		for (int x = 0; x < i; x++) {
			Thread t = new Thread(new Test(countDown));
			t.start();
		}

	}
}
```

　　执行结果如下：   
>ready : Thread-0   
>ready : Thread-1   
>ready : Thread-2   
>ready : Thread-3   
>ready : Thread-4   
>ready : Thread-5   
>ready : Thread-6   
>ready : Thread-7   
>enter : Thread-0   
>enter : Thread-7   
>enter : Thread-6   
>enter : Thread-5   
>enter : Thread-4   
>enter : Thread-3   
>enter : Thread-2   
>enter : Thread-1   

　　除了线程-0，其它线程都是按LIFO顺序执行，由于线程-0是第一个准备竞争锁，那时锁还没被其它线程占用。因此，\_cxq是LIFO队列。   

　　2.synchronized和lock的性能比较。这个好像在本次讨论的话题之外，既然写了就把代码放上来。代码主要模拟在多线程竞争的条件下，对比synchronized和lock的消耗时间。   

```java
package io.github.peikun.multithread;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class SynchronizedVsLockTest {
	static final Object mutex1 = new Object();
	static final Object mutex2 = new Object();
	static final Lock lock1 = new ReentrantLock();
	static final Lock lock2 = new ReentrantLock();
	static int counter1 = 0;
	static int counter2 = 0;
	static final AtomicInteger counter3 = new AtomicInteger();
	static final AtomicInteger counter4 = new AtomicInteger();
	public static final int LOOPS = 50 * 1000 * 1000;

	public static void main(String[] args) throws InterruptedException {
		testSvL1();
		// testSvL2();
	}

	static void testSvL1() throws InterruptedException {
		int[] threadCountArray = { 1, 2, 4, 8, 16, 32, 64 };
		for (final int t : threadCountArray) {
			doTest(t, new Runnable() {
				public void run() {
					for (int i = 0; i < (LOOPS / t); i++) {
						synchronized (mutex1) {
							counter1++;
						}
					}
				}

				@Override
				public String toString() {
					return "1x synchronized {}";
				}
			});
			doTest(t, new Runnable() {
				public void run() {
					for (int i = 0; i < (LOOPS / t); i++) {
						lock1.lock();

						try {
							counter1++;
						} finally {
							lock1.unlock();
						}
					}
				}

				@Override
				public String toString() {
					return "1x Lock.lock()/unlock()";
				}
			});
			doTest(t, new Runnable() {
				public void run() {
					for (int i = 0; i < (LOOPS / t); i++) {
						counter3.getAndIncrement();
					}
				}

				@Override
				public String toString() {
					return "1x AtomicInteger";
				}
			});
		}
	}

	/*
	 * static void testSvL2() throws InterruptedException { for (final int t :
	 * new int[] { 1, 2, 4, 8, 16, 32, 64 }) { doTest(t, 3, new Runnable() {
	 * 
	 * @Override public void run() { for (int i = 0; i < (LOOPS / t); i++) {
	 * synchronized (mutex1) { counter1++; }
	 * 
	 * synchronized (mutex2) { counter2++; } } }
	 * 
	 * @Override public String toString() { return "2x synchronized {}"; } });
	 * doTest(t, 4, new Runnable() {
	 * 
	 * @Override public void run() { for (int i = 0; i < (LOOPS / t); i++) {
	 * lock1.lock();
	 * 
	 * try { counter1++; } finally { lock1.unlock(); }
	 * 
	 * lock2.lock();
	 * 
	 * try { counter2++; } finally { lock2.unlock(); } } }
	 * 
	 * @Override public String toString() { return "2x Lock.lock()/unlock()"; }
	 * }); doTest(t, 5, new Runnable() {
	 * 
	 * @Override public void run() { for (int i = 0; i < (LOOPS / t); i++) {
	 * counter3.getAndIncrement(); counter4.getAndIncrement(); } }
	 * 
	 * @Override public String toString() { return "2x AtomicInteger"; } }); } }
	 */

	private static void doTest(int threads, Runnable runnable)
			throws InterruptedException {
		long start = System.currentTimeMillis();

		ExecutorService es = Executors.newFixedThreadPool(threads);
		try {
			for (int i = 0; i < threads; i++)
				es.execute(runnable);
		} catch (Exception e) {
			throw new AssertionError(e);
		} finally {
			es.shutdown();
		}

		es.awaitTermination(1, TimeUnit.MINUTES);

		long time = (System.currentTimeMillis() - start);
		System.out.printf("%s with %d threads took %.3f seconds%n",
				runnable.toString(), threads, time / 1e3);
	}
}
```

　　执行结果如下：
>1x synchronized {} with 1 threads took 0.698 seconds   
>1x Lock.lock()/unlock() with 1 threads took 1.063 seconds   
>1x AtomicInteger with 1 threads took 0.441 seconds   
>1x synchronized {} with 2 threads took 5.856 seconds   
>1x Lock.lock()/unlock() with 2 threads took 1.111 seconds   
>1x AtomicInteger with 2 threads took 0.442 seconds   
>1x synchronized {} with 4 threads took 5.949 seconds   
>1x Lock.lock()/unlock() with 4 threads took 1.030 seconds   
>1x AtomicInteger with 4 threads took 0.404 seconds   
>1x synchronized {} with 8 threads took 5.688 seconds   
>1x Lock.lock()/unlock() with 8 threads took 1.046 seconds   
>1x AtomicInteger with 8 threads took 0.421 seconds   
>1x synchronized {} with 16 threads took 5.883 seconds   
>1x Lock.lock()/unlock() with 16 threads took 1.032 seconds   
>1x AtomicInteger with 16 threads took 0.400 seconds   
>1x synchronized {} with 32 threads took 5.756 seconds   
>1x Lock.lock()/unlock() with 32 threads took 1.097 seconds   
>1x AtomicInteger with 32 threads took 0.465 seconds   
>1x synchronized {} with 64 threads took 5.781 seconds   
>1x Lock.lock()/unlock() with 64 threads took 1.031 seconds   
>1x AtomicInteger with 64 threads took 0.432 seconds   

　　可以看到，synchronized在多线程竞争的情况下，执行时间相对比较长。   

附录：   
[Java synchronized 等待顺序](http://www.whoslab.me/blog/?p=393 "Java synchronized 等待顺序")   
[JVM底层又是如何实现synchronized的？](http://wenku.baidu.com/view/41480552f01dc281e53af090.html "JVM底层又是如何实现synchronized的？")   
[hotspot内部对synchronize的实现](http://blog.hummingbird-one.com/?p=10086 "hotspot内部对synchronize的实现")   
[探索Java同步机制](https://www.ibm.com/developerworks/cn/java/j-lo-synchronized/ "探索 Java 同步机制")   
[ObjectMonitor.cpp源码](http://searchcode.com/codesearch/view/17981004 "ObjectMonitor.cpp源码")
