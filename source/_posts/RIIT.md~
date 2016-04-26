---
title: 我所理解的java反射
date: 2016-04-20 19:24:29
tags: java reflection
---
## 为什么要使用反射
   在面向对象的编程中，多态是最常用的概念。基本上，面向对象的思想之所以能这么流行，能出现多种灵活的设计模式，多态的特征是功不可没的。多态，使我们将抽象和具体隔离。使得父类给出接口，子类具体实现。降低了编程的复杂性。但在某些情况下，在我们将子类向上转型后，有希望知道这个类的具体类型，和操作某些子类特有的行为。这时候，反射能帮上我们的忙。
   在某些情况了，你新的类在你的程序编译好很久后才会出现。比如：你从互联网上下载一段代码，你明确知道，这段代码代表的是一个类。可是，你怎么才能很使用它呢。反射就是我们用于解决这种问题的工具。

## Class

一切都是对象,是java的基本设计思想。在我们编写每一个.java文件后，编译器会将我们的.java文件编译成.class文件。当我们调用name.class的静态方法时，jvm的类加载器会将我们的class文件加载进内存。这个从侧面证实了，一个类的构造函数也是静态函数，虽然他们有static关键字。class也是一个对象。我们可以利用这个对象。来创建“常规”的对象.从上面的描述，我们也可以了解到，java是动态加载的语言。当类首次被引用的时候，才会被加载进内存。这点c++中就很难做到。

{%codeblock%}
1、Class.forname(className);  //可以不是使用对象，拿到这个类的Class引用。
2、Class name = name.class;   //类字面常量生成Class引用。在编译时就会受到检查
{%endcodeblock%}
值得注意的一点，使用方法1获得类是的引用时，其静态成员会被初始化。
使用方法2时，其静态成员只有在其类的静态成员第一次被使用时，才会被初始化。

 泛型和Class注意点
 假设 存在 父类 Father
 子类 Child extends father
{%codeblock%}
Class<Child> c=Child.classm
{%endcodeblock%}
因为编译时就知道c.getSuperclass()得到的不只是Father这个类，更明确到他是Child的父类。

## isInstance 和 isInstanceOf
isInstanceOf和isInstance这两个方法都是用来确定对象的类型。
但是用起来有一些差别。总体来说，isInstanceOf实在编译期间就能明确对象类型的。而isInstance实在运行期间才能确定。
用法也稍微也不同
{%codeblock%}
A a = new A();
a.instanceOf A; //true
a.getClass().instanceOf(A);//true
{%endcodeblock%}
值得注意的是，isInstance比较影响效率，在能使用isInstanceOf 的情况下，尽可能的使用isInstanceOf
## 动态代理
代理模式事实上就是在具体实现类中间加一个中间层。把具体实现隔离开来。
在java中，出来能实现我们经常见到的代理模式，我们还能通过实现接口InvocationHanler来实现动态代理。

{%codeblock%}
public List getList(final List list){
    return (List) Proxy.newProxyInstance(DummyProxy.class.getClassLoader(), new Class[] { List.class}
                                 new InvocationHanler(){
                                         public Object invoke(Object proxy, Method method Object[] args) throws Throwable{
                                             if("add".equals(method.getName())){
                                                 throw new UnsupportdOperationException();
                                             }else{
                                                 return method.invoke(list,args);
                                             }
                                         }
                                     }                               );}
{%endcodeblock%}
上面例子是执行List.class的方法，如果遇到add方法，则抛出异常。剩下的方法正常执行。
例子来自（http://www.infoq.com/cn/articles/cf-java-reflection-dynamic-proxy）
## 反射的危害
事实上，反射是很强大的。但是伴随而来的是权限方面的难以管理。原则上来说，反射只要知道方法名，就能调用此方法。private关键字也起不到保护的作用。
但是，有趣的是，final域相对是安全的，运行是，修改它，系统并不会抛出异常，但是事实上它的值并没有被修改。
