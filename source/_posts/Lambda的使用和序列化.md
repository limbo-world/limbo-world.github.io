---
title: Lambda的使用和序列化
copyright: true
date: 2022-03-11 19:26:53
tags:
---

最近想给orm工具包增加Lambda支持，实现`User:getId`映射到表`id`字段，碰到了问题，进行总结。

<!-- more -->

# 匿名内部类

> java7的时候，有时候为了方便都使用内部类来实现部分代码

## 函数式接口定义
```java
// @FunctionalInterface 不加也行，会自动推导
public interface MyFunction<T, R> {
    R apply(T t);
}
```
## 匿名内部类使用
```java
public class MyTest {

    public static  <T, R> void execute(MyFunction<T, R> f, T a) {
        f.apply(a);
    }

    public static void main(String[] args) {
        MyTest.execute(new MyFunction<User, String>() {
            @Override
            public String apply(User user) {
                return user.getName();
            }
        }, new User());
    }
}
```
编译一下
```bash
javac *.java
```
反编译
```java
> $ javap -c -p MyTest
public class MyTest {
   // ......
   public static <T, R> void execute(MyFunction<T, R>, T);
    Code:
       0: aload_0
       1: aload_1
       2: invokeinterface #2,  2            // InterfaceMethod MyFunction.apply:(Ljava/lang/Object;)Ljava/lang/Object;
       7: pop
       8: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #3                  // class MyTest$1
       3: dup
       4: invokespecial #4                  // Method MyTest$1."<init>":()V
       7: new           #5                  // class User
      10: dup
      11: invokespecial #6                  // Method User."<init>":()V
      14: invokestatic  #7                  // Method execute:(LMyFunction;Ljava/lang/Object;)V
      17: return
}
```
在main函数中看到创建了匿名内部类`MyTest$1`

```java
> $ javap -c -p MyTest\$1
final class MyTest$1 implements MyFunction<User, java.lang.String> {
  // ......
  public java.lang.String apply(User);
    Code:
       0: aload_1
       1: invokevirtual #2                  // Method User.getName:()Ljava/lang/String;
       4: areturn

  public java.lang.Object apply(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: checkcast     #3                  // class User
       5: invokevirtual #4                  // Method apply:(LUser;)Ljava/lang/String;
       8: areturn
}
```
# Lambda的使用

> “Lambda 表达式”(lambda expression)是一个匿名函数，其基于数学中的λ演算得名。不像js可以var a = function，由于java中不能脱离类，通常以一个匿名内部类+内部一个方法构成的。

## 使用Lambda简化匿名内部类使用
```java
// 修改main方法代码如下
MyTest.execute(User::getName, new User());
```
重新编译
```bash
javac MyTest.java MyFunction.java  User.java
```
看看字节码
```java
> $ javap -v -p MyTest
......
Constant pool:
	// ......
    #3 = InvokeDynamic      #0:#29         // #0:apply:()LMyFunction;
	//......
{
  // ......

public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: invokedynamic #3,  0              // InvokeDynamic #0:apply:()LMyFunction;
         5: new           #4                  // class User
         8: dup
         9: invokespecial #5                  // Method User."<init>":()V
        12: invokestatic  #6                  // Method execute:(LMyFunction;Ljava/lang/Object;)V
        15: return
      LineNumberTable:
        line 8: 0
        line 9: 15
}
SourceFile: "MyTest.java"
InnerClasses:
     public static final #47= #46 of #52; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #25 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #26 (Ljava/lang/Object;)Ljava/lang/Object;
      #27 invokevirtual User.getName:()Ljava/lang/String;
      #28 (LUser;)Ljava/lang/String;
```
从结果来看会调用LambdaMetafactory.metafactory -> buildCallSite() -> spinInnerClass()。spinInnerClass此方法在内存创建类

> 使用 java -Djdk.internal.lambda.dumpProxyClasses MyTest 会看到生成 MyTest$$Lambda$1.class

```java
> $ javap -p MyTest\$\$Lambda\$1
final class MyTest$$Lambda$1 implements MyFunction {
  private MyTest$$Lambda$1();
  public java.lang.Object apply(java.lang.Object);
}
```

# Lambda序列化

## 实现Serializable接口

由`MfTest$$Lambda$1`可以知道，此类未实现Serializable无法序列化，所以修改MyFunction如下

```java
public interface MyFunction<T, R> extends Serializable {
    R apply(T t);
}
```

加上继承序列化后查看

```java
final class MyTest$$Lambda$1 implements MyFunction {
	......
    private final Object writeReplace() {
        return new SerializedLambda(MyTest.class, "MyFunction", "apply", "(Ljava/lang/Object;)Ljava/lang/Object;", 5, "User", "getName", "()Ljava/lang/String;", "(LUser;)Ljava/lang/String;", new Object[0]);
    }
}
```
可以看到实现了Serializable接口会增加`writeReplace`方法。
```java
// 这个时候反编译MfTest结果如下
public class MyTest {
	// ......

  private static java.lang.Object $deserializeLambda$(java.lang.invoke.SerializedLambda);
    // .......
}
```
可以看到这里多了`$deserializeLambda$`方法。从SerializedLambda类上的注释可以了解到，SerializedLambda为可序列化lambdas的实现者，可以序列化的lambda类实现了writeReplace方法返回SerializedLambda的一个实例。
SerializedLambda有一个readResolve方法，它在捕获类中寻找名为$deserializeLambda$(SerializedLambda)的静态方法，将其作为第一个参数调用，并返回结果。

# 获取映射属性

## 打印Lambda类所有方法

```java
MyFunction<User, String> func = User::getName;
Class lambdaClass = func.getClass();
//打印类名：
System.out.print("类名：");
System.out.println(lambdaClass.getName());
//打印接口名：
System.out.print("接口名：");
Arrays.stream(lambdaClass.getInterfaces()).forEach(System.out::print);
System.out.println();
//打印方法名：
System.out.println("方法名：");
for (Method method : lambdaClass.getDeclaredMethods()) {
    System.out.println(method.getName() + "  ");
}
```

输出如下
```
类名：lambda.STest$$Lambda$1/999661724
接口名：interface lambda.MyFunction
方法名：
apply  
writeReplace
```
从这里没有突破口

## SerializedLambda属性
```java

public final class SerializedLambda implements Serializable {
    private static final long serialVersionUID = 8025925345765570181L;
    // lambda表达式出现的类
    private final Class<?> capturingClass;
    // 返回lambda对象的静态类型的名称，斜杠分隔形式
    private final String functionalInterfaceClass;
    // lambda工厂站点当前的函数接口方法的名称
    private final String functionalInterfaceMethodName;
    // 在lambda工厂站点上显示的函数接口方法的签名
    private final String functionalInterfaceMethodSignature;
    // 用斜杠分隔的形式表示，用于存放实现方法的类
    private final String implClass;
    // 实现方法的名称
    private final String implMethodName;
    // 实现方法的签名
    private final String implMethodSignature;
    // 方法处理实现方法的kind
    private final int implMethodKind;
    // 在类型变量被它们从捕获站点的实例化替换后，主功能接口方法的签名
    private final String instantiatedMethodType;
    // lambda工厂站点的动态参数，表示由lambda捕获的变量
    private final Object[] capturedArgs;

    public SerializedLambda(Class<?> capturingClass,
                            String functionalInterfaceClass,
                            String functionalInterfaceMethodName,
                            String functionalInterfaceMethodSignature,
                            int implMethodKind,
                            String implClass,
                            String implMethodName,
                            String implMethodSignature,
                            String instantiatedMethodType,
                            Object[] capturedArgs) {
        ......
    }
}
```

从writeReplace的代码可以知道，我们要找的其实是`getName`这个方法名，只要能获取到这个方法，我们就可以获取到对应的属性名称。从这个入参可以知道对应的是`implMethodName`属性，所以我们需要获取SerializedLambda再调用getImplMethodName获取对应的方法名称。
```java
return new SerializedLambda(MyTest.class, "MyFunction", "apply", "(Ljava/lang/Object;)Ljava/lang/Object;", 5, "User", "getName", "()Ljava/lang/String;", "(LUser;)Ljava/lang/String;", new Object[0]);
```

## 获取SerializedLambda实例

### 反射方式获取

```java
MyFunction<User, String> func = User::getName;
Class lambdaClass = func.getClass();
Method method = lambdaClass.getDeclaredMethod("writeReplace");
method.setAccessible(Boolean.TRUE);
SerializedLambda serializedLambda = (SerializedLambda) method.invoke(func);
String getterMethod = serializedLambda.getImplMethodName();
System.out.println("lambda表达式调用的方法名：" + getterMethod);
String fieldName = Introspector.decapitalize(getterMethod.replace("get", ""));
System.out.println("根据方法名得到的字段名：" + fieldName);

// 输出
lambda表达式调用的方法名：getName
根据方法名得到的字段名：name
```

### 序列化/反序列化获取

整体流程可参考下图
{% asset_image lambda-serializability.jpeg %}

```java
MyFunction<User, String> func = User::getName;
Class<?> clazz = func.getClass();
// 将流转为 SerializedLambda 对象
ByteArrayOutputStream baos = new ByteArrayOutputStream(1024);
try (ObjectOutputStream oos = new ObjectOutputStream(baos)) {
    oos.writeObject(func);
    oos.flush();
} catch (IOException ex) {
    throw new IllegalArgumentException("Failed to serialize object of type: " + clazz, ex);
}

try (ObjectInputStream objIn = new ObjectInputStream(new ByteArrayInputStream(baos.toByteArray())) {
    @Override
    protected Class<?> resolveClass(ObjectStreamClass objectStreamClass) throws IOException, ClassNotFoundException {
        Class<?> clazz;
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        try {
            clazz = Class.forName(objectStreamClass.getName(), false, cl);
        } catch (Exception ex) {
            clazz = super.resolveClass(objectStreamClass);
        }
        return clazz;
    }
}) {
    SerializedLambda lambda = (SerializedLambda) objIn.readObject();
    System.out.println(lambda.getImplMethodName());
} catch (ClassNotFoundException | IOException e) {
    throw new IllegalArgumentException("This is impossible to happen", e);
}

// 输出
Exception in thread "main" java.lang.ClassCastException: lambda.STest$$Lambda$4/114935352 cannot be cast to lambda.SerializedLambda
```

执行上面代码会报类型无法转换的问题。这是由于ObjectInputStream.readObject()方法会最终回调SerializedLambda.readResolve()方法，导致返回的结果是一个实现Serlializable的Lambda表达式实例，在实际操作的时候无法转成SerializedLambda实例。当前获取到的实例如下

{% asset_image LambdaObject.png %}

> 所以这里需要中断这个调用提前返回结果，参考Mybatis plus的实现方式，我们拷贝SerializedLambda定义一个新的类，然后删除readResolve()方法。

```java
// 修改代码
return clazz == java.lang.invoke.SerializedLambda.class ? lambda.SerializedLambda.class : clazz;

// 输出
getName
```
{% asset_image SerializedLambdaObject.jpg %}

--- 
参考文档：
1. [JDK中Lambda表达式的序列化与SerializedLambda的巧妙使用](https://www.cnblogs.com/throwable/p/15611586.html)
2. [Lambda表达式实现方式](https://blog.csdn.net/zxhoo/article/details/38495085)
3. Mybatis-plus 源码
