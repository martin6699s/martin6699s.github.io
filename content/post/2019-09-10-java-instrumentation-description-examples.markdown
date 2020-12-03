---
layout: post
title:  JAVA Instrumentation之浅谈
date:   2019-09-10 13:32:20 +0300
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: post-1.jpg # Add image post (optional)
tags: [Blog, Java Instrument]
author: martin6699s # Add name author (optional)
---

instrumentation是JAVA SE 5 引进来的新特性，它可以构建一个独立于应用程序的代理程序，用于监控和协助运行在JVM上的程序，甚至能够替换和修改某些类的定义（这类功能被叫做打桩或插桩，另外如ASM、javasisst也是打桩、插桩），实现JVM级别的AOP，不过只能在程序启动之前javaagent指定代理程序。到JAVA SE 6，对instrument功能进行了增强，还可以支持程序启动后的 instrument、本地代码（native code）instrument，以及动态改变 classpath 等等，具体能做的如：内存dump，线程dump，类信息统计（比如加载的类及大小以及实际个数），非侵入式运行时AOP（btrace及阿里出品的jvm-sandbox就是使用了instrument），动态设置vm flag(但没有几个可以用的，因为有些flag都是在jvm启动过程中使用，是一次性的)，打印vm flag，获取系统属性等。



## Instrumentation 的基本功能和用法

`java.lang.instrument`包的具体实现，依赖于JVMTI。JVMTI（Java Virtual Machine Tool Interface）是一套由Java虚拟机提供的，为JVM相关的工具提供的本地编程接口集合。JVMTI能做什么呢？它支持第三方工具程序以代理的方式连接和访问JVM（实际上就是进程间socket通信），并利用JVMTI提供的丰富的编码接口，完成很多跟JVM相关的功能。

Instrumentation有两种方式：

1. premain方式，在程序启动之前通过`-javaagent`指定代理程序; （Java se 5）
2. agentmain方式，在程序启动之后指定代理程序;（Java se 6）



### premain方式

#### （1）说明

在Java SE 5当中，开发者可以让Instrumentation代理在main函数运行前执行。这种方式可以能够替换和修改某些类的定义，增加方法、删除方法，修改方法等。实现具体步骤如下：

1. 代理程序编写premain函数

      在代理程序中编写一个Java类，包含以下两个方法之一：

   ```
   （1） public static void premain(String agentArgs, Instrumentation inst);
   （2） public static void premain(String agentArgs);
   ```

      其中，如果两个方法同时存在，优先执行方法（1），忽略方法（2）

   在premain方法内完成对类的各种操作，如替代或修改 。实际上就是修改或代替原有的类字节码。另外在premain方式下，JVM在执行main方法之前，先会调用代理程序的premain方法，其中inst参数是由JVM自动传入。

   

2. 配置代理应用的MANIFEST.MF文件

   代理程序需要配置META-INF目录下MANIFEST.MF，配置内容如下：

   ```
   Manifest-Version: 1.0
   Premain-Class: Premain
   Can-Redefine-Classes: true
   Can-Retransform-Classes: true
   Boot-Class-Path：javassist.jar
   ```

   Premain-Class： 指定了代理使用的类，也就是，该类含有premain方法。当一个代理在JVM中启动时会从MANIFEST.MF找到对应的代理类。如果有命名空间，应该包含命名空间；

   Can-Redefine-Classes:  指明代理JAR文件是否可以重新定义类。可选，默认为false。使用代理，需要设为true；

   Boot-Class-Path:  指明该代理JAR文件中有用到的第三方jar包放到这里，例如代理要用到javassist.jar来操作操作字节码，就需要指定，并在打包的时候把Boot-Class-Path指定的jar包也一起打包；

   Can-Retransform-Classes: 它的作用在于只有在代理JAR文件中`Can-Retransform-Classes`设置了 manifest属性 `true`并且JVM支持此功能时，才支持重新转换。在单个JVM的单个实例化期间，对此方法的多次调用将返回相同结果。默认为false。在agentmain方式下需要将该属性设为true。
   
3. 代理程序 jar文件打包

   需要把含premain代理程序、META-INF目录及文件、依赖包一同打包成jar。如打包成MyAgent.jar

4. 运行

   `java  -javaagent: /path/to/martin6699s/MyAgent.jar  application.jar`

#### （2）premain方式举例：

随便建个项目命名为mainTest， 在mainTest项目中建一个类`TimeHelloWorld`，等会利用代理修改该类方法：

```java
package com.myagent.martin6699s;

/**
 * @author martin
 * @date 2019/9/9
 **/
public class TimeHelloWorld {

    public void sayHello() {
        try {
            Thread.sleep(2000);
            System.out.println("hello world!!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void sayHelloMartin() {
        try {
            Thread.sleep(1000);
            System.out.println("hello martin!!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

再建一个含main方法的类`Main`作为程序入口，创建TimeHelloWorld并执行sayHello方法

```java
package com.myagent.martin6699s;

/**
 * @author martin
 * @date 2019/9/11
 **/
public class Main {

    public static void main(String[] args) throws InterruptedException {

        System.out.println("开始------");

        for(int i = 0; i < 1; i++) {
            TimeHelloWorld helloWorld = new TimeHelloWorld();
            helloWorld.sayHello();
            helloWorld.sayHelloMartin();

            Thread.sleep(1000);
        }

        System.out.println("结束------");
    }
}

```

直接运行Main类，输出如下：

```
开始------
hello world!!
hello martin!!
结束------
```

现在需要计算sayHello和sayHelloMartin方法的执行时间，但又不想在这两方法内部加代码；

那执行main方法前把TimeHelloWorld类字节码替换成我们想要的字节码，

那接下来再建一个名为MyAgent的项目, 在MyAgent项目中建一个PremainTransformer类实现ClassFileTransformer接口，其中实现transform方法来完成字节码的替换。这里用到javassist进行字节码操作。

```java
package com.myagent.martin6699s;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import javassist.CtNewMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.ArrayList;
import java.util.List;

/**
 * @author martin
 * @date 2019/9/9
 **/
public class PremainTransformer implements ClassFileTransformer {

    final static String prefix = "\nlong startTime = System.currentTimeMillis();\n";

    final static String postfix = "\nlong endTime = System.currentTimeMillis();\n";
    final static List<String> methodList = new ArrayList<String>();


    public PremainTransformer() {
       methodList.add("com.myagent.martin6699s.TimeHelloWorld.sayHello");
       methodList.add("com.myagent.martin6699s.TimeHelloWorld.sayHelloMartin");
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {

        if(className.startsWith("com/myagent/martin6699s")) {
            // 判断加载的类包名是不是我们需要目标类的包名
            className = className.replaceAll("/", ".");

            CtClass ctClass = null;

            try {

                ctClass = ClassPool.getDefault().get(className);

                for(String method: methodList) {

                    if(method.startsWith(className)) {

                        String methodName = method.
                                substring(method.lastIndexOf(".") + 1, method.length());

                        String outputStr = "\nSystem.out.println(\"方法执行时间:  "
                                + methodName
                                + "\" + (endTime - startTime) + \"ms.\");";

                        CtMethod ctMethod = ctClass
                                .getDeclaredMethod(methodName);

                        String newMethodName = methodName + "$impl";

                        ctMethod.setName(newMethodName);

                        //创建新的方法，复制原来的方法 ，名字为原来的名字
                        CtMethod newMethod = CtNewMethod.copy(ctMethod, methodName, ctClass,null);

                        // 构建新的方法体
                        StringBuilder bodyStr = new StringBuilder(250);
                        bodyStr.append("{");
                        bodyStr.append(prefix);
                        bodyStr.append(newMethodName+"($$);\n");
                        bodyStr.append(postfix);
                        bodyStr.append(outputStr);
                        bodyStr.append("}");

                        newMethod.setBody(bodyStr.toString());
                        ctClass.addMethod(newMethod);
                    }
                }

                System.out.println("准备替换-----" + className);
                return ctClass.toBytecode();

            } catch (Exception e) {
                e.printStackTrace();

                System.out.println("异常信息：" + e.getMessage());
            }
        }


        return null;
    }
}

```



然后，需要在代理程序MyAgent中在建一个类，类中有premain方法: 

```
package com.myagent.martin6699s;

import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

/**
 * @author martin
 * @date 2019/9/11
 **/
public class MyPremainAgent {

    public static void premain(String agentArgs, Instrumentation inst)
            throws ClassNotFoundException, UnmodifiableClassException {

        System.out.println("进入---premain---start");
        // 加转换类,可以加多个
        inst.addTransformer(new PremainTransformer());
        System.out.println("进入---premain---end");
    }
}

```



在MyAgent项目中建META-INF/MANIFEST.MF文件，MANIFEST.MF文件内容：

```
Manifest-Version: 1.0
Premain-Class: com.myagent.martin6699s.MyPremainAgent
Can-Redefine-Classes: true
Boot-Class-Path: javassist.jar
```

然后把MyAgent项目连同javassist.jar、MANIFEST.MF一起打包成MyAgent.jar，到放到`E:\MyAgent\target\`目录下

把mainTest项目两个类编译成.class文件输出`E:\Work\MultiProject\JAVASE6Agent\mainTest\target\production\mainTest`目录下

最后就可以执行
```
 java -javaagent:E:\MyAgent\target\MyAgent.jar -Dfile.encoding=UTF-8  -classpath "E:\Work\MultiProject\JAVASE6Agent\mainTest\target\production\mainTest" com.myagent.martin6699s.Main
```



输出如下

```
进入---premain---start
进入---premain---end
准备替换-----com.myagent.martin6699s.Main
开始------
准备替换-----com.myagent.martin6699s.TimeHelloWorld
hello world!!
方法执行时间:  sayHello2000ms.
hello martin!!
方法执行时间:  sayHelloMartin1000ms.
结束------
```



### Agentmain方式

#### （1）说明

agentmain方式是JAVA SE 6对Instrumentation的增强，支持程序启动后动态代理对类的转换，使用retransformClasses方法声明待转换的类，转换过程不会导致其初始化，静态变量的值保持不变，但它也有局限性，下面段话引自`JAVA SE 7 文档中关于Instrumentation restransformClasses方法介绍`:

>
>
>The retransformation may change method bodies, the constant pool and attributes. The retransformation must not add, remove or rename fields or methods, change the signatures of methods, or change inheritance. These restrictions maybe be lifted in future versions. The class file bytes are not checked, verified and installed until after the transformations have been applied, if the resultant bytes are in error this method will throw an exception.
>
>If this method throws an exception, no classes have been retransformed.

翻译过来就是：转换可以改变方法主体、常量池和属性。转换不能新增、删除、重命名字段和方法，不能改变方法签名，不能改变继承关系。在将来的版本可能会取消限制。在转换之前，不会对类文件字节码进行检查、检验、安装，如果转换后合成的字节码发生错误，restransformClasses方法将抛异常。 如果restransformClasses方法抛异常，将不会进行类转换，也就是不会改变类字节码。

因为这个局限性，导致之前运行代码是出现异常：

`Caused by: java.lang.UnsupportedOperationException: class redefinition failed: attempted to add a method` 

原因是：我在AgentmainTransformer类里面使用了和premain方式一样的代码，在类中构建了一个新的方法，导致异常，后来改成替换方法主体就通过了。

具体步骤：

1. 编写含agentmain方法的类及实现ClassFileTransformer的类；
2. 编写META-INF/MANIFEST.MF文件；
3. 打包jar；
4. 编写控制程序模拟监控，实际上就是为了初始化并触发Attach Listener监听事件进程，动态代理替换原先字节码；
5. 运行监控程序；

#### （2）Agentman方式举例

和premain一样同样需要mainTest项目来作为目标项目，但稍微改一下Main类，把for循环 `i < 1`改为`i < 60` 方便测试：

```java
package com.myagent.martin6699s;

/**
 * @author martin
 * @date 2019/9/11
 **/
public class Main {

    public static void main(String[] args) throws InterruptedException {

        System.out.println("开始------");

        for(int i = 0; i < 60; i++) {
            TimeHelloWorld helloWorld = new TimeHelloWorld();
            helloWorld.sayHello();
            helloWorld.sayHelloMartin();

            Thread.sleep(10000);
        }

        System.out.println("结束------");
    }
}
```



接下来编写AgentmainTransformer类，和PremainTransformer类不同的是直接修改方法体，而不是创建方法体副本（相当于创建新方法）

```java
package com.myagent.martin6699s;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.util.ArrayList;
import java.util.List;

/**
 * @author martin
 * @date 2019/9/9
 **/
public class AgentmainTransformer implements ClassFileTransformer {

    final static List<String> methodList = new ArrayList<String>();


    public AgentmainTransformer() {
        methodList.add("com.myagent.martin6699s.TimeHelloWorld.sayHello");
        methodList.add("com.myagent.martin6699s.TimeHelloWorld.sayHelloMartin");

    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {

        if(className.startsWith("com/myagent/martin6699s")) {
            // 判断加载的类包名是不是我们需要目标类的包名
            className = className.replaceAll("/", ".");

            CtClass ctClass = null;

            try {

                ctClass = ClassPool.getDefault().get(className);

                for(String method: methodList) {

                    if(method.startsWith(className)) {

                        String methodName = method.
                                substring(method.lastIndexOf(".") + 1);

                        CtMethod ctMethod = ctClass
                                .getDeclaredMethod(methodName);
                        ctMethod.addLocalVariable("startTime", CtClass.longType);
                        ctMethod.insertBefore("startTime = System.currentTimeMillis();");
                        ctMethod.insertAfter("{final long endTime = System.currentTimeMillis();\n System.out.println(\"方法("+ className +"#" + methodName +")执行时间: \" + (endTime-startTime) + \" 毫秒(ms) \");}");

                    }
                }

                System.out.println("准备替换-----" + className);
                return ctClass.toBytecode();

            } catch (Exception e) {
                e.printStackTrace();
                System.out.println("异常信息：" + e.getMessage());
            }
        }


        return null;
    }
}

```



含有agentmain方法的类：

```
package com.myagent.martin6699s;

import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

/**
 * @author martin
 * @date 2019/9/9
 **/
public class MyAgentmainAgent {

   public static void agentmain(String agentArgs, Instrumentation inst)
       throws ClassNotFoundException, UnmodifiableClassException {

       String targetClassPath = "com.myagent.martin6699s.TimeHelloWorld";

       for(Class<?> clazz : inst.getAllLoadedClasses()) {

           if(!inst.isModifiableClass(clazz)) {
               continue;
           }

           if(clazz.getName().equals(targetClassPath)) {
               inst.addTransformer(new AgentmainTransformer(), true);
               inst.retransformClasses(clazz);
               System.out.println("Agent Main Done");

           }
       }

   }


}

```



然后编写MANIFEST.MF配置文件：

```
Manifest-Version: 1.0
Agent-Class: com.myagent.martin6699s.MyAgentmainAgent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Boot-Class-Path: javassist.jar
```

其中Agent-Class指明代理jar中代理执行代码在那个类上

然后打包成MyAgent2.jar

为了运行 Attach API，我们需要一个控制程序来模拟监控过程：

```
import com.sun.tools.attach.AgentInitializationException;
import com.sun.tools.attach.AgentLoadException;
import com.sun.tools.attach.AttachNotSupportedException;
import com.sun.tools.attach.VirtualMachine;

import java.io.IOException;

/**
 * @author martin
 * @date 2019/9/11
 **/
public class AttachTreadMain {


    public static void main(String[] args) {

        String pid = args[0];
        VirtualMachine vm = null;

        System.out.println("pid=" + pid);
        try {
             vm = VirtualMachine.attach(pid);
             vm.loadAgent("E:\\Work\\MultiProject\\JAVASE6Agent\\MyAgent2\\target\\MyAgent2.jar");
        } catch (AttachNotSupportedException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (AgentLoadException e) {
            e.printStackTrace();
        } catch (AgentInitializationException e) {
            e.printStackTrace();
        } finally {
            if(vm != null) {
                try {
                    vm.detach();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

其中loadAgent方法参数为MyAgent2.jar的文件绝对路径。

最后第二步，运行目标项目mainTest的Main类，通过`jps -l`找到Main类程序对应的pid是多少，假如pid=10012

最后把pid作为程序参数传给监控程序AttachTreadMain并运行。结果目标项目mainTest输出结果如下：

```
开始------
hello world!!
hello martin!!
hello world!!
hello martin!!
hello world!!
hello martin!!
hello world!!
hello martin!!
准备替换-----com.myagent.martin6699s.TimeHelloWorld
Agent Main Done
hello world!!
方法(com.myagent.martin6699s.TimeHelloWorld#sayHello)执行时间: 2001 毫秒(ms) 
hello martin!!
方法(com.myagent.martin6699s.TimeHelloWorld#sayHelloMartin)执行时间: 1000 毫秒(ms) 
......
```



##### 坑点

agentmain方式需要指定Can-Retransform-Classes为true，否则会报异常`Caused by: java.lang.UnsupportedOperationException: adding retransformable transformers is not supported in this environment`



到此是不是觉得这个功能很强大，以后就可以线上调试不用重启服务了，是不是很开森！

其实Intellij idea 调试、强大的Btrace bug跟踪调试工具、阿里出品的jvm-sandbox都是利用JAVA这种特性来做的。



示例代码放在github上了： https://github.com/martin6699s/java-examples/tree/master/Instrumentation



【参考链接】

 [IBM Java SE 6 特性 Instrumentation功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html) 

[Java SE 7 Instrumentation文档](https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/Instrumentation.html)

[Java SE 7 VirtualMachine文档](https://docs.oracle.com/javase/7/docs/jdk/api/attach/spec/com/sun/tools/attach/VirtualMachine.html)

[java agent机制](https://blog.51cto.com/5162886/2044541)

[入门科普，围绕JVM的各种外挂技术](http://calvin1978.blogcn.com/articles/vjtools-tools4.html)

[JavaAgent源码分析](https://blog.csdn.net/jinxin70/article/details/85690904)