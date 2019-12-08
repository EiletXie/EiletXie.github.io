---
title: JVM内存溢出Demo与JDK版本的关联
date: 2019-09-16 13:12:22
tags:
- JVM
toc: true

---

# JVM内存溢出Demo与JDK版本的关联



​         最近看经典的JVM书籍，周志明的《深入理解JAVA虚拟机》第三版，看到很多不错JVM内存溢出示例，我这里拿JDK1.6和JDK1.8版本进行了Demo的测试和总结。

通过该文章，您可以了解相关知识

- 不同类型的内存溢出应该是在哪种情形下，哪块区域的问题
- 相关的JVM参数设置，调优
- 相同的代码、不同JDK版本造成内存溢出的类型不同



先列出内存溢出的几种类型：

- java.lang.OutOfMemoryError: Java heap space  堆内存溢出
- java.lang.OutOfMemoryError: PermGen space 永久代溢出
- java.lang.StackOverflowError 栈溢出
- java.lang.OutOfMemoryError: Metaspace  元空间溢出
- java.lang.OutOfMemoryError  直接内存溢出

<!--more-->
<br/>

## 栈溢出

### java.lang.StackOverflowError

栈溢出，发生在方法相互调用过多，或者递归无限调用时。


#### JVM参数

**-Xss   栈内存容量**


#### 产生的情形

class中方法的调用是以栈帧的方式存储在 虚拟机栈中，当栈的深度不足，或者栈帧过大时就会导致虚拟机栈抛出异常。我们这就用递归，当递归无终止条件就会产生 java.lang.StackOverflowError
<br/>

#### 示例代码

```java
public class StackDemo3 {

	private int stackDepth = 1;
	
	public  void test1(){
		stackDepth++;
		test1();
	}
	
	public static void main(String[] args) {
		StackDemo3 s = new StackDemo3();
		try {
			s.test1();
		} catch (Throwable e) {
			System.out.println("stack depth: " + s.stackDepth);
			throw e;
		}
	}

}
```

​    {% asset_img stack2.jpg  jvm栈参数图 %}

#### 运行报错

  {% asset_img stack.jpg  jvm栈异常图 %}

<br/>

### java.lang.OutOfMemoryError

栈溢出的另外一种异常

**这种是发生在虚拟机扩展栈时无法申请到足够的内存空间**

例如： 方法中有while(true)不break的情况时

由于JDK1.8 无法限制JAVA线程中方法的内存扩展大小，所以运行到一半我怕我电脑死机停掉了。。

```java
public class StackDemo4 {

	private void noStop(){
		while(true){
		}
	}
	
	private void stackRunNoStop(){
		// 加速扩展栈方法占用的内存
		while(true){
			Thread thread = new Thread(new Runnable() {
				@Override
				public void run() {
					noStop();
				}
			});
		}
	}
	
	public static void main(String[] args) {
		StackDemo4 s = new StackDemo4();
		s.stackRunNoStop();
	}

}
```
<br/>
## 堆空间溢出

### java.lang.OutOfMemoryError: Java heap space

堆空间是存放对象实例的地方 new Object()

#### JVM参数

- **-Xmx  最大分配内存，默认为物理内存的 "1 / 4"**

- **-Xms  设置初始分配大小，默认为物理内存的 "1 / 64"**

- **-XX:+PrintGCDetails     输出详细的GC处理日志**

  <br/>

#### 产生的情形

​      对象的实例存放在堆空间中，那么我们只需要不停的产生字符串对象即可。

#### 示例代码

```java
public class HeapDemo {

	public static void main(String[] args) {
		// 设置JVM参数为几M大小
		String str = "test HeapOutOfMemory";
		
		// 无限生产新的str对象
		while(true){
			str += str + new Random().nextInt(66666666) + new Random().nextInt(99999999);
		}
	}

}
```
<br/>
#### 运行报错

  {% asset_img heap.jpg  堆空间异常图 %}

 <br/>

## 运行时常量池溢出

这块区域是内存溢出类型最多的，为什么呢？

- JDK1.6及之前的版本:  常量池在方法区，方法区属于永久代，会报出 OutOfMemoryError: PermGen space
- JDK1.7:  常量池是位于堆空间中，它会报 java.lang.OutOfMemoryError
- JDK1.8: 方法区和运行时常量池位于元空间中，它报的是元空间溢出

由于我电脑上只安装了 JDK1.8和JDK1.6，这里只做这两种版本的示例。
<br/>

### java.lang.OutOfMemoryError: Metaspace

**元空间溢出**

​         JDK1.8版本提出元空间这个概念，它和新生代、老年代组成JVM内存区域，方法区和运行时常量池在JDK1.8版本中放置在了元空间中，默认元空间使用的是本地内存。

#### JVM参数

- -XX:MetaspaceSize 初始空间大小，达到该值就会触发垃圾收集进行类型卸载
- -XX:MaxMetaspaceSize，最大空间，默认是没有限制的

设置元空间大小 -XX:MetaspaceSize=5M  -XX:MaxMetaspaceSize=10M

#### 产生的情形

​         String.intern() 是一个 Native 方法，它的作用是：如果字符串常量池中包含该String对象的常量池中，并且返回该String对象的引用。 

​        我们用一个list作为常量池的引用，防止被GC，然后无限插入新的字符串

#### 示例代码

```java
public class RuntimeConstantPoolOOM {

	public static void main(String[] args) {
		// 用List保证常量池引用，避免被FULL GC回收常量池行为
		List<String> list = new ArrayList<>();
		int i = 0;
		
		while(true){
		    list.add(String.valueOf(i++).intern());
		}
	}

}
```

#### 运行报错

  {% asset_img pool.jpg  常量池异常图 %}

### java.lang.OutOfMemoryError: PermGen space

#### JVM参数

在JDK1.6及之前的版本中，由于常量池分配在永久代内，我们可以通过-XX:PermSize和-XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量，你可以给个10M大小



产生情形和示例代码是一致的，区别只是JDK版本不同，常量池所在的内存区域发生变化，从而导致不同内存溢出异常。

#### 运行报错

 {% asset_img pool2.jpg  常量池异常1.6图 %}

## 方法区溢出

方法区抛出的异常与常量池一致，一个包含关系

#### 产生的情形

由于方法区用于存放Class的相关信息：如类名、访问修饰符、常量池

所以测试的话，思路就是 运行程序时产生大量的类填满的类去填满方法区，直到溢出。

这里用CGLib直接操作字节码运行时产生大量的动态类。



刚开始设置的是 方法区的大小，发现没什么效果。

​     **VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M**

​         由于JDK1.8时 已经去除了永久代，方法区在元空间中，而元空间未设置的话使用的是本地内存，所以我测试一直未报溢出。

直到设置的JVM参数  元空间的大小才报元空间溢出的异常

​      -XX:MetaspaceSize=5M  -XX:MaxMetaspaceSize=5M

### java.lang.OutOfMemoryError: Metaspace

#### 示例代码

```java
/**
 * 借助CGLib方法区出现内存溢出异常 
 * @author XC-Eilet
 *
 *         CGLIB是一个强大、高性能的字节码生成库，它用于在运行时扩展Java类和实现接口；本质上它是通过动态的生成一个子类去覆盖所要代理的类（
 *         非final修饰的类和方法）。Enhancer是一个非常重要的类，它允许为非接口类型创建一个JAVA代理，
 *         Enhancer动态的创建给定类的子类并且拦截代理类的所有的方法，和JDK动态代理不一样的是不管是接口还是类它都能正常工作。
 * 
 *         net.sf.cglib.proxy.Callback接口：在cglib包中是一个很关键的接口，所有被net.sf.cglib.proxy
 *         .Enhancer类调用的回调（callback）接口都要继承这个接口。
 *         net.sf.cglib.proxy.MethodInterceptor接口:是通用的回调（callback）类型，
 *         他经常被AOP用来实现拦截（intercept）方法的调用；
 */
public class JavaMethodAreaOOM {

	public static void main(String[] args) {
		while (true) {
			/*
			 * Enhancer可能是CGLIB中最常用的一个类，和Java1.3动态代理中引入的Proxy类差不多(如果对Proxy不懂，可以参考这里)。
			 * 和Proxy不同的是，Enhancer既能够代理普通的class，也能够代理接口。
			 */
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(OOMObject.class);
			enhancer.setUseCache(false);
			enhancer.setCallback(new MethodInterceptor() {
				
				public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
					return proxy.invokeSuper(obj, args);
				}
				
			});
			enhancer.create();
		}
	}

	static class OOMObject {
		
	}
}

```

#### 运行报错

 {% asset_img method.jpg  方法区异常图 %}

### java.lang.OutOfMemoryError: PermGen space
<br/>

JDK1.6版本

用JDK1.6 版本测试时，由于限制了方法区大小，就开始报永久代错误了。

#### JVM参数

设置方法区大小

-XX:PermSize=10M -XX:MaxPermSize=10M

代码与上面示例代码保持一致

#### 运行报错

 {% asset_img method2.jpg  方法区异常图 %}

<br/>

## 本机直接内存溢出

### java.lang.OutOfMemoryError

#### JVM参数

DirectMemory 容量通过-XX:MaxDirectMemorySize 指定，如果不指定的话，默认等于Java堆的最大值（-Xmx）一样。

这里用 反射获取Unsafe实例进行直接内存分配。

#### 示例代码

```java
/**
 * VM Args: -Xmx20M -XX:MaxDirectMemorySize=10M
 * @author XC-Eilet
 */
public class DirectMemoryOOM {

	private static final int _1MB = 1024*1024;
	public static void main(String[] args) throws Exception {
      Field unsafeField = Unsafe.class.getDeclaredFields()[0];
      unsafeField.setAccessible(true);
      Unsafe unsafe = (Unsafe) unsafeField.get(null);
      while(true){
    	  unsafe.allocateMemory(_1MB);
      }
	}

}
```

#### 运行报错

 {% asset_img direct.jpg  直接内存异常图 %}