
---
title: Java 如何有效地避免OOM：善于利用软引用和弱引用
date: 2020-3-5
tags:
	- Java引用
categories: Java
---

## 一.了解 强引用、软引用、弱引用、虚引用的概念

在Java中，虽然不需要程序员手动去管理对象的生命周期，但是如果希望某些对象具备一定的生命周期的话（比如内存不足时JVM就会自动回收某些对象从而避免OutOfMemory的错误）就需要用到软引用和弱引用了。

　　从Java SE2开始，就提供了四种类型的引用：强引用、软引用、弱引用和虚引用。Java中提供这四种引用类型主要有两个目的：第一是可以让程序员通过代码的方式决定某些对象的生命周期；第二是有利于JVM进行垃圾回收。下面来阐述一下这四种类型引用的概念：

<!-- more -->

### 1.强引用（StrongReference）

强引用就是指在程序代码之中普遍存在的，比如下面这段代码中的object和str都是强引用：

```java
Object object = new Object();
String str = "hello";
```

只要某个对象有强引用与之关联，JVM必定不会回收这个对象，即使在内存不足的情况下，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。

　　如果想中断强引用和某个对象之间的关联，可以显示地将引用赋值为null，这样一来的话，JVM在合适的时间就会回收该对象。

　　比如ArrayList类的clear方法中就是通过将引用赋值为null来实现清理工作的：

```java
/**
 * Removes all of the elements from this list.  The list will
 * be empty after this call returns.
 */
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

### 2.软引用（SoftReference）

　　软引用是用来描述一些有用但并不是必需的对象，在Java中用java.lang.ref.SoftReference类来表示。对于软引用关联着的对象，只有在内存不足的时候JVM才会回收该对象。因此，这一点可以很好地用来解决OOM的问题，并且这个特性很适合用来实现缓存：比如网页缓存、图片缓存等。

　　软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被JVM回收，这个软引用就会被加入到与之关联的引用队列中。下面是一个使用示例：

```java
import java.lang.ref.SoftReference;
 
public class Main {
    public static void main(String[] args) {
         
        SoftReference<String> sr = new SoftReference<String>(new String("hello"));
        System.out.println(sr.get());
    }
}
```
### 3.弱引用（WeakReference）

　　弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在java中，用java.lang.ref.WeakReference类来表示。下面是使用示例：

```java
import java.lang.ref.WeakReference;
 
public class Main {
    public static void main(String[] args) {
     
        WeakReference<String> sr = new WeakReference<String>(new String("hello"));
         
        System.out.println(sr.get());
        System.gc();                //通知JVM的gc进行垃圾回收
        System.out.println(sr.get());
    }
}
```

输出结果为：

```java
hello
null
```
　　
第二个输出结果是null，这说明只要JVM进行垃圾回收，被弱引用关联的对象必定会被回收掉。不过要注意的是，这里所说的被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。

　　弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被JVM回收，这个软引用就会被加入到与之关联的引用队列中。

### 4.虚引用（PhantomReference）

　　虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用java.lang.ref.PhantomReference类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。

　　要注意的是，虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
 
 
public class Main {
    public static void main(String[] args) {
        ReferenceQueue<String> queue = new ReferenceQueue<String>();
        PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);
        System.out.println(pr.get());
    }
}
```

## 弱引用应用

比如Android中的MVP模式开发的时候,Presenter层需要引用View(Activity),此处就可以使用弱引用,这样当Activity被回收后,Presenter层的弱引用就会被回收就不会造成内存泄露了

当然在activity没有被回收的时候,也就是Presenter层中的弱引用还有强引用引用着activity,所以垃圾回收的时候不会回收弱引用,之后activity被回收,activity强引用释放后弱引用经历gc的时候才会被回收

[谁动了我的Activity：一个新启动创建的 Activity 对象到底被谁引用了？](https://blog.csdn.net/weixin_44339238/article/details/103924373)

#### 延伸

由于Presenter经常性地需要执行一些耗时操作，如网络请求。而Presenter持有了Activity的强引用，如果在请求结束之前Activity被销毁了，那么由于网络请求还没有返回，导致Presenter一直持有Activity对象，使得Activity对象无法被回收，此时就发生了内存泄漏。

为了解决上述问题，可以通过弱引用和Activity，Fragment的生命周期来解决这个问题。首先建立一个Presenter抽象，它是一个泛型类，泛型类型为View角色要实现的接口类型，代码如下：

```java
public abstract class BasePresenter<T> {
    protected Reference<T> mViewRef;//View接口类型的弱引用
    
    public void attachView(T view){
        mViewRef = new WeakReference<T>(view);//建立关联
    }
    
    //获取View
    protected T getView(){
        return mViewRef.get();
    }
    
    //判断是否与View建立了关联
    public boolean isViewAttached(){
        return mViewRef!=null&&mViewRef.get()!=null;
    }
    
    //解除关联
    public void detachView(){
        if(mViewRef!=null){
            mViewRef.clear();
            mViewRef = null;
        }
    }
}
```

然后，创建一个MVPBaseActivity基类，通过这个基类的生命周期来控制它和Presenter的关系，代码如下：

```java
public abstract class MVPBaseActivity<V, P extends BasePresenter<V>> extends Activity {
    /**
    *Presenter对象
    */
    protected P mPresenter;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //创建Presenter
        mPresenter = createPresenter();
        mPresenter.attachView((V) this);
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
         /**
        *弱引用防止内存溢出
        */
        mPresenter.detachView();
    }
    
    /**
    *子类负责提供具体的Presenter对象
    */
    protected abstract P createPresenter();
}
```

MVPBaseActivity含有两个泛型参数，第一个是View层接口，第二个是Presenter的具体类型。通过泛型参数，使得一些通用的逻辑可以被抽象到MVPBaseActivity类中。
