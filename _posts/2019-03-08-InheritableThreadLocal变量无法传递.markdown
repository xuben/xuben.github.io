---
layout: post
title:  "InheritableThreadLocal变量无法传递"
date:   2019-03-08 17:29:40 +0800
categories: Java
---
# 背景知识
我们都知道InheritableThreadLocal类型的变量可以在父子线程之间传递。具体实现为：每个线程都维护了`threadLocals`和`inheritableThreadLocals`成员变量，线程在初始化时会根据父线程的`inheritableThreadLocals`创建自身的`inheritableThreadLocals`。具体的创建由ThreadLocal类负责实现。

线程的`threadLocals`和`inheritableThreadLocals`成员变量都是ThreadLocal.ThreadLocalMap类型，这个类在功能上类似于HashMap，是ThreadLocal或者InheritableThreadLocal类型变量的集合。

Thread.java
```
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;

/*
 * InheritableThreadLocal values pertaining to this thread. This map is
 * maintained by the InheritableThreadLocal class.
 */
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

`init`是线程的初始化方法，注意这里省略了无关的一些代码

Thread.java
```
/**
 * Initializes a Thread.
 *
 * @param g the Thread group
 * @param target the object whose run() method gets called
 * @param name the name of the new Thread
 * @param stackSize the desired stack size for the new thread, or
 *        zero to indicate that this parameter is to be ignored.
 * @param acc the AccessControlContext to inherit, or
 *            AccessController.getContext() if null
 * @param inheritThreadLocals if {@code true}, inherit initial values for
 *            inheritable thread-locals from the constructing thread
 */
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    ……
    Thread parent = currentThread();
    ……
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    ……
}
```

# 问题描述
笔者最近在使用InheritableThreadLocal时无意中碰到了一个问题。先来看我自定义的ThreadLocal类型，这个类的默认初始值为当前时间戳。

MyThreadLocal.java
```
import java.util.Date;

public class MyThreadLocal extends InheritableThreadLocal<Long> {

	@Override
	protected Long initialValue() {
		return new Date().getTime();
	}
}
```

在这里我创建了一个子线程myThread，然后打印当前线程的ThreadLocal变量的值，过2秒后启动子线程，由子线程打印这个ThreadLocal变量的值。

MyThread.java
```
public class MyThread extends Thread {

	public static ThreadLocal<Long> tl = new MyThreadLocal();

	@Override
	public void run() {
		System.out.println(tl.get());
	}

	public static void main(String args[]) {
		MyThread myThread = new MyThread();
		System.out.println(tl.get());

		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

		myThread.start();
	}
}
```

由于MyThreadLocal继承了InheritableThreadLocal，因此我期望两者可以打印出相同的结果。然而无论运行多少遍这个程序，实际结果总是相差2秒以上（非常接近2秒）。由此可见这个ThreadLocal变量并没有从父线程传递到子线程。

# 问题分析
笔者试着通过分析源代码来查找原因。可以看到线程中的`threadLocals`变量默认为null，而且Thread类中并没有任何对其赋值的地方，事实上通过注释也可以发现`threadLocals`和`inheritableThreadLocals`都是由ThreadLocal类维护的

ThreadLocal类的`createMap`方法为线程初始化`threadLocals`变量

ThreadLocal.java
```
/**
 * Create the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the map
 */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

InheritableThreadLocal类的`createMap`方法为线程初始化`inheritableThreadLocals`变量

InheritableThreadLocal.java
```
/**
 * Create the map associated with a ThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the table.
 */
void createMap(Thread t, T firstValue) {
    t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
}
```

而调用`createMap`方法的地方为ThreadLocal的`set`方法和`setInitialValue`方法，后者被`get`方法调用。ThreadLocal本身的构造函数并没有做任何事情，即使我们通过重写`initialValue`方法设置默认值。也就是说，如果我们创建了ThreadLocal类型的对象后没有调用`set`方法设置初始值或者调用`get`方法设置默认的初始值，那么这个对象不会被添加到线程的`threadLocals`或者`inheritableThreadLocals`中。此时创建的子线程也不能获取到父线程的InheritableThreadLocal类型的变量。由此导致了InheritableThreadLocal变量无法在父子线程之间传递。
当然解决方法也很简单，在子线程创建之前调用`set`方法设置变量的值或者调用`get`方法获取默认值即可。

# 结论
InheritableThreadLocal类型的变量要在子线程创建之前设置初始值，否则不能在父子线程之间共享。
