---
layout: post
title: java对象拷贝
category: java
tags: [java]
---

###什么是对象拷贝

对象，可以简单理解为是内存区域的一段数据。对象拷贝就是把这段数据复制到内存中的另一段区域，与原来的区别是，内存地址不同。

###为什么进行对象拷贝

由于对象其实是引用，在一个对象有很多引用的情况下，修改对象的属性，有很高的潜在风险。为了规避风险，提高程序确定性，拷贝是个很好的选择。


###java中的clone、cloneable

####clone

方法签名：
    
    protected Object clone() throws CloneNotSupportedException

* java.lang.Object类的一个实例方法，protected, 不能直接访问

* 通过 field-by-field assignment 方式产生一个新的对象，不会去调用构造函数

* 返回值是一个Object对象，不能直接使用，需要进行 type cast

* 如果需要调用此方法，被调用类必须实现 Cloneable 接口

对于任何一个对象x，clone方法遵循如下规约：

    x.clone() != x  // always true
    
    x.clone().getClass() == x.getClass() // true, but these are not absolute requirements
    
    x.clone().equals(x) // be true, this is not an absolute requirement

以下是一个利用clone实现对象拷贝的例子：

	public class X implements Cloneable {
        public Object clone() {
            try {
           	    return super.clone();
        	} catch (CloneNotSupportedException e) {
           	    throw new InternalError(e.toString()); 
            }
        }
	}
 
####cloneable

* 一个没有任何method的接口，仅用来向编译器表名，一个class是否能被拷贝

* 改变了java.lang.Object.clone()的行为，本来是protected的方法，却能以public的方式调用

####clone方法存在的问题

clone 仅仅是 field-by-field assignment，如果 field 是一个对象引用，那么赋值到新对象的也是一个引用。如何实现对象属性也能被拷贝呢？为了区别 Object.clone 的实现方式，引入一个新的概念，<font color="red"><em>Deep clone</em></font>，而 Object.clone 的实现方式叫 <font color="red"><em>Shallow clone</em></font>。两者的区别如下图：

![浅拷贝](/assets/images/java-object-copy-1.jpg)  ![深拷贝](/assets/images/java-object-copy-2.jpg) 


以下是一个利用clone实现深拷贝的例子：

	public class X  implements Cloneable {
		private int a; // primitive
		private String b; // reference to immutable object
		private Y c; // where class Y has mutator methods

		public Object clone() { // deep clone
			try {
				X other = (X) super.clone(); // fields a & b OK
				other.c = (Y) c.clone(); // fix c by making a copy
				return other; // return the deep clone
			} catch (CloneNotSupportedException e) {
				throw new InternalError(e.toString());
			}
		}
	}

####对象拷贝最佳实践

* 不要使用java.lang.Object.clone()

* 使用拷贝构造函数

		public Y(Y y);

* 使用静态工厂方法

		public static Y newInstance(Y y);

###参考

1、[Effective Java - Override clone() method judiciously](https://www.slideshare.net/fmshaon/effective-java-override-clone-method-judiciously)

2、[Deep Copy And Shallow Copy](http://www.jusfortechies.com/java/core-java/deepcopy_and_shallowcopy.php)

3、[clone() Method Of java.lang.Object Class](http://javaconceptoftheday.com/clone-method-java-lang-object-class/)
