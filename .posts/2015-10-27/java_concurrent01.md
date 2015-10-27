<!--{layout:default title:Java Concurrent并发工具(一)}-->

最近打算好好看看java.util.concurrent里面的源码，正好总结一下常用的并发工具类，一级它们的用法和原理，记录在此。

**java.util.concurrent这个Java并发工具包是在Java1.5时被添加进去的。使用这个工具包，当我们在编写多线程的java代码时，能更方便/语义更明确的解决线程安全问题。**

###java.util.concurrent.locks.Lock

在Java1.5之前，多线程同步控制是靠synchronized关键字来实现同步方法/同步代码块；在1.5之后，我们可以使用java.util.concurrent.locks.Lock接口以及多个实现类来进行线程同步了。

<pre class="language-java line-numbers">
<code>
//Lock接口部分代码
public interface Lock{
	/**
	 * 加锁
	 */
	void lock();
	/**
	 * 释放锁
	 */
	void unlock();
}
</code>
</pre>

Lock的常用实现类有ReentrantLock(重入锁)和ReentrantReadWriteLock(重入读写锁)，重入锁就像普通的synchronized，每次只能有一个线程获得锁，其他线程阻塞等待锁释放，并且同一把锁，线程可以获取多次；读写锁则可以多个线程获取读锁，只有一个线程获取写锁；

<pre class="language-java line-numbers">
<code>
package com.raven.lock;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

//重入锁的用法
public class LockTest {
	static final Lock lock = new ReentrantLock();
	static volatile int i = 0; 
	public static void main(String[] args) {
		for(int j=0;j<50;j++){
			new Thread(new Runnable() {	
				@Override
				public void run() {
					lock.lock();		//加锁
					System.out.println("value:"+(i++));
					lock.unlock();		//释放锁
				}
			}).start();
		}
	}
}
</code>
</pre>

正确的使用锁了，可以保证上面多个线程按照正确的顺序输出变量i的值

具体锁的实现原理，这里暂时不分析，准备以后单独写一篇文章详细说明

###java.util.concurrent.locks.Condition

锁或者synchronized只是保证避免多个线程修改共享状态时的状态不确定问题，如果需要控制进程的执行顺序，那么就需要用到wait/notify(notifyAll)或者java.util.concurrent.locks.Condition了

不确定的ping-pong顺序

<pre class="language-java line-numbers">
<code>
public class Main {
	private final static Object lock = new Object();
	static class MyTask implements Runnable{
		private String msg;
		public MyTask(String msg){
			this.msg = msg;
		}
		private void say(){
			synchronized(lock){
				//System.out.println并不保证线程安全，需要加锁，或者同步代码块
				//但是也不能保证ping->pong->ping->pong...这样正确的顺序
				System.out.println(this.msg);
			}
		}
		@Override
		public void run() {
			for(int i=0;i<3;i++){
				try {
					Thread.sleep(1);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
				this.say();
			}
		}
	}
	public static void main(String [] args){
		final Object lock = new Object();
		Thread t1=new Thread(new MyTask("ping"));
		Thread t2=new Thread(new MyTask("pong"));
		t1.start();
		t2.start();
	}
}
</code>
</pre>

使用wait/notify保证一致的顺序(两个线程交替输出ping pong)

<pre class="language-java line-numbers">
<code>
public class Main {
	/**
	 * 用一个状态表示当前应该运行的线程(从ping开始)
	 */
	private static String flag = "ping";
	
	static class MyTask implements Runnable {
		//其他略，只改进了say方法
		private void say(){
			synchronized(lock){
				try {
					//如果当前线程不该运行
					//那么wait，阻塞线程，让出锁
					if(!flag.equals(this.msg)){
						lock.wait();
					} 
					System.out.println(this.msg);
					//修改状态
					if("ping".equals(flag)){
						flag = "pong";
					} else {
						flag = "ping";
					}
				} catch (InterruptedException e) {
					e.printStackTrace();
				} finally {
					//通知其他处于wait态的线程
					//notify会随机激活一个处于wati的线程
					//lock.notify();
					//notifyAll会全部激活wait线程
					//线程自己从新竞争锁
					lock.notifyAll();
				}
			}
		}
	}
}
</code>
</pre>

上面使用wait/notify来保证ping-pong顺序，下面我们使用java.util.concurrent.locks.Condition来实现同样的功能

<pre class="language-java line-numbers">
<code>
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

public class Main{
	/**
	 * 用一个状态表示当前应该运行的线程(从ping开始)
	 */
	private static String flag = "ping";
	
	private final static Lock lock = new ReentrantLock();
	private final static Condition condition = lock.newCondition();
	
	static class MyTask implements Runnable {
		//其他略，只改进了say方法
		private void say() {
			//用lock代替synchronized
			lock.lock();
			try {
				if (!flag.equals(this.msg)) {
					//用condition.await()代替wait
					condition.await();
				}
				System.out.println(this.msg);
				if ("ping".equals(flag)) {
					flag = "pong";
				} else {
					flag = "ping";
				}
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} finally {
				//用condition.signal(All) 代替 notify(All)
				//condition.signal();
				condition.signalAll();
			}
			lock.unlock();
		}
	}
}
</code>
</pre>

###Java原子操作和Atomic系列类

>原子操作指的是在一步之内就完成而且不能被中断。原子操作在多线程环境中是线程安全的，无需考虑同步的问题。在java中，下列操作是原子操作：

>1. all assignments of primitive types except for long and double(除了long和double以外的所有基本类型赋值操作，double和long的赋值不是原子的)
>2. all assignments of references(所有类实例的引用赋值操作)
>3. all operations of java.concurrent.Atomic* classes(所有java.concurrent.atomic.Atomic*类的操作)
>4. all assignments to volatile longs and doubles(所有volatile的long和double型变量赋值操作)

Atomic是Java提供的一系列用来简化同步处理的原子类，比如在Java1.5之前，我们要实现一个线程安全的计数器类，大概是这样的
<pre class="language-java line-numbers">
<code>
public class Counter {
	//volatile只保证变量的更新对线程立即可见，但是不保证操作的原子性，如++
	//要保证原子性，还需要自己控制同步
	private volatile int count;
	
	public Counter(){
		this.count = 0;
	}
	
	public Counter(int initialValue){
		this.count = initialValue;
	}
	
	public int get(){
		return this.count;
	}
	
	/**
	 * 递增，并且返回递增的值
	 */
	public synchronized int increment(){
		this.count++;
		return this.count;
	}
}
</code>
</pre>

如果使用java.util.concurrent.atomic.AtomicInteger类实现同样的逻辑，则会简单很多：
<pre class="language-java line-numbers">
<code>
import java.util.concurrent.atomic.AtomicInteger;
public class Counter{
	private AtomicInteger count;
	
	public Counter(){
		this.count = new AtomicInteger(0);
	}
	
	public Counter(int initialValue){
		this.count = new AtomicInteger(initialValue);
	}
	
	public int get(){
		return this.count.get();
	}
	
	/**
	 * 递增，并且返回递增的值
	 */
	public int increment(){
		//AtomicInteger类给我们提供了线程的方法
		return this.count.incrementAndGet();
	}
}
</code>
</pre>

Atomic系列类中最重要的方法是boolean compareAndSet(except, update)方法，也就是java中实现Lock-Free的是CAS(比较与交换，Compare and Swap)算法；

使用AtomicBoolean，我们可以基于CAS算法实现一个乐观锁：

<pre class="language-java line-numbers">
<code>
package com.raven.lock.impl;

import java.util.concurrent.atomic.AtomicBoolean;

import com.raven.lock.Lock;

public class SimpleLock implements Lock {
	//冲突次数
	private AtomicBoolean status = new AtomicBoolean(false);
	@Override
	public void lock() {
		//自旋
		while(!status.compareAndSet(false, true)){
			//当compareAndSet返回false时，线程会进入死循环（自旋）达到竞争锁的目的
			//compareAndSet返回false，说明已经有其他线程修改了status的值，不再是false了
			//获得锁成功的线程进行了compareAndSet操作，更新了status的值为true
			//乐观锁不会真正的阻塞线程，而是让线程进入死循环(自旋状态)
		}
	}

	@Override
	public void unlock() {
		//释放锁只需要将status改为false
		//其他在自旋状态的线程会退出死循环
		//从而获得锁
		status.set(false);
	}
}
</code>
</pre>

-------

总结：

* 多线程互斥(synchronized Lock)和同步(wait/notify Condition)操作是Java多线程中常用到的操作；
* volatile并不能保证操作的原子性，因此不能用来控制线程互斥同步；
* Atomic*保证了原子操作，可以用CAS实现乐观锁；
* 乐观锁和悲观锁要结合实际情况来选择，自旋浪费的cpu时间不一定比锁的切换带来的消耗少！

-------

相关文章：

* [很早之前总结的《Java线程同步互斥》](../2015-07-01/java_sync_mut.html)