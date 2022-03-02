---
type: docs 
title: "Java Agent"
linkTitle: "Java Agent"
weight: 5
---

在平时的开发中，我们不可避免的会使用到Debug工具，JVM作为一个单独的进程，我们使用的Debug工具可以获取JVM运行时的相关的信息，查看变量值，甚至加入断点控制，还有我们平时使用JDK自带的JMAP、JSTACK等工具，可以在JVM运行时动态的dump内存、查询线程信息，甚至一些第三方的工具，比如说京东内部使用的JEX、pfinder，阿里巴巴的Arthas，优秀的开源的框架skywalking等等，也可以做到这些，那么这些工具究竟是通过什么技术手段来实现对JVM的监控和动态修改呢？本文会进行介绍和简单的原理分析，同时附带一些样例代码来进行分析。

## 从 JVMTI 说起

JVM在设计之初，就考虑到了虚拟机状态的监控、debug、线程和内存分析等功能，在 JDK5.0 之前，JVM 规范就定义了 JVMPI(Java Virtual Machine Profiler Interface)也就是 JVM 分析接口以及 JVMDI(Java Virtual Machine Debug Interface)也就是 JVM 调试接口，JDK5 以及以后的版本，这两套接口合并成了一套，也就是 Java Virtual Machine Tool Interface，就是我们这里说的 JVMTI，这里需要注意的是：

- JVMTI 是一套 JVM 的接口规范，不同的 JVM 实现方式可以不同，有的 JVM 提供了拓展性的功能，比如 openJ9，当然也可能存在 JVM 不提供这个接口的实现。
- JVMTI 提供的是 Native 方式调用的 API，也就是常说的 JNI 方式，JVMTI 接口用 C/C++ 的语言提供，最终以动态链接库的形式由 JVM 加载并运行。

使用 JNI 方式调用 JVMTI 接口访问目标虚拟机的大体过程入下图:

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224022.png" style="display:block;margin-left:auto;margin-right:auto;width:80%;" alt="20220221224022" /></div>

jvmti.h 头文件中定义了 JVMTI 接口提供的方法，但是其方法的实现是由 JVM 提供商实现的，比如说 hotspot 虚拟机其实现大部分在 src\share\vm\prims\jvmtiEnv.cpp 这个文件中。

## Instrument Agent

在 Jdk1.5 之后，Java 语言中开始提供 Instrumentation 接口(java.lang.instrument)让开发者可以使用 Java 语言编写 Agent，但是其根本实现还是依靠 JVMTI，只不过是 SUN 在工具包(sun.instrument.InstrumentationImpl)编写了一些 native 方法，并且然后在 JDK 里提供了这些 native 方法的实现类(jdk\src\share\instrument\JPLISAgent.c)，最终需要调用 jvmti.h 头文件定义的方法，跟前文提到采用 JNI 方式访问 JVMTI 提供的方法并无差异，大体流程如下图：

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224152.png" style="display:block;margin-left:auto;margin-right:auto;width:80%;" alt="20220221224152" /></div>

但是 Instrument agent 仅使用到了 JVMTI 提供部分功能，对开发者来说，主要提供的是对 JVM 加载的类字节码进行插桩操作。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224232.png" style="display:block;margin-left:auto;margin-right:auto;width:70%;" alt="20220221224232" /></div>

## 1. **JVM启动时Agent**

我们知道，JVM 启动时可以指定 -javaagent:xxx.jar 参数来实现启动时代理，这里 xxx.jar 就是需要被代理到目标 JVM 上的 JAR 包，实现一个可以代理到指定 JVM 的 JAR 包需要满足以下条件：

- JAR 包的 MANIFEST.MF 清单文件中定义 Premain-Class 属性，指定一个类，加入 Can-Redefine-Classes 和 Can-Retr ansform-Classes 选项。
- JAR 包中包含清单文件中定义的这个类，类中包含 premain 方法，方法逻辑可以自己实现。

了解到这两点，我们可以定义下列类：

```java
import java.lang.instrument.Instrumentation;

public class AgentMain {

    // JVM启动时agent
    public static void premain(String args, Instrumentation inst) {
        agent0(args, inst);
    }

    public static void agent0(String args, Instrumentation inst) {
        System.out.println("agent is running!");
        // 添加一个类转换器
        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(ClassLoader loader, 
                                    String className, 
                                    Class<?> classBeingRedefined, 
                                    ProtectionDomain protectionDomain, 
                                    byte[] classfileBuffer) {
                // JVM加载的所有类会流经这个类转换器
                // 这里找到自定义的测试类
                if (className.endsWith("WorkerMain")) {
                    System.out.println("transform class WorkerMain");
                }
                // 直接返回原本的字节码
                return classfileBuffer;
            }
        });
    }
}
```

JAR 包内对应的清单文件(MANIFEST.MF)需要有如下内容：

```
PreMain-Class: AgentMain
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

-javaagent 所指定 jar 包内 Premain-Class 类的 premain 方法，方法签名可以有两种：

1. `public static void premain(String agentArgs, Instrumentation inst)`

2. `public static void premain(String agentArgs)`

JVM 会优先加载 1 签名的方法，加载成功忽略 2，如果 1 没有，加载 2 方法。这个逻辑在 sun.instrument.InstrumentationImpl 类中实现。

需要说明的是，addTransformer 方法的作用是添加一个字节码转换器，这个方法的入参对象需要实现 ClassFileTransformer 接口，唯一需要实现的方法就是 transform 方法，这个方法可以用来修改加载类的字节码，目前我们并不对字节码进行修改。

最后定义测试类：package test;

```java
import java.util.Random;

class WorkerMain {

    public static void main(String[] args) throws InterruptedException {
        for (; ; ) {
            int x = new Random().nextInt();
            new WorkerMain().test(x);
        }
    }

    public void test(int x) throws InterruptedException {
        Thread.sleep(2000);
        System.out.println("i'm working " + x);
    }
}
```

启动时添加 -javaagent:xxx.jar 参数，指定 agent 刚刚生成的JAR包，可以看到运行结果：

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224659.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221224659" /></div>

### 流程解析

下面尝试结合 JDK 源码对该流程进行浅析：

JVM 开始启动时会解析 -javaagent 参数，如果存在这个参数，就会执行 Agent_OnLoad 方法读取并解析指定 JAR 包后生成 JPLISAgent 对象，然后注册 jvmtiEventCallbacks.VMInit 这个事件，也就是虚拟机初始化事件，并设置该事件的回调函数 eventHandlerVMInit，这些代码逻辑在 jdk\src\share\instrument\InvocationAdapter.c 和  jdk\src\share\instrument\JPLISAgent.c 中实现。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224755.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221224755" /></div>

在 JVM 初始化时会调用之前注册的 eventHandlerVMInit 事件的回调函数，进入 processJavaStart 这个函数，首先会在注册另一个 JVM 事件 ClassFileLoadHook，然后会真正的执行我们在 Java 代码层面编写的 premain 方法。当 JVM 开始装载类字节码文件时，会触发之前注册的 ClassFileLoadHook 事件的回调方法 eventHandlerClassFileLoadHook，这个回调函数调用 transformClassFile 方法，生成新的字节码，被 JVM 装载，完成了启动时代理的全部流程。

以上代码逻辑在 jdk\src\share\instrument\JPLISAgent.c 中实现。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221224843.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221224843" /></div>

## 2. **JVM运行时Agent**

在 JDK1.6 版本中，SUN 更进一步，提供了可以在 JVM 运行时代理的能力，和启动时代理类似，只需要满足：

- JAR 包的 MANIFEST.MF 清单文件中定义 Agent-Class 属性，指定一个类，加入 Can-Redefine-Classes 和 Can-Retransform-Classes 选项。
- JAR 包中包含清单文件中定义的这个类，类中包含 agentmain 方法，方法逻辑可以自己实现。

运行时 Agent 可以在 JVM 运行时动态的修改某个类的字节码，然后 JVM 会重定义这个类（不需要创建新的类加载器），但是为了保证 JVM 的正常运行，新定义的类相较于原来的类需要满足：

1. 父类是同一个。

2. 实现的接口数也要相同，并且是相同的接口。

3. 类访问符必须一致。

4. 字段数和字段名要一致。

5. 新增或删除的方法必须是 private static/final 的。 

6. 可以修改方法内部代码。

运行时 Agent 需要借助 JVM 的 Attach 机制，简单来说就是 JVM 提供的一种通信机制，JVM 中会存在一个 Attach Listener 线程，监听其他 JVM 的 attach 请求，其通信方式基于 socket，JVM Attach 机制大体流程图如下：

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221225028.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221225028" /></div>

### JVM Attach

SUN 在 JDK 中提供了 Attach 机制的 Java 语言工具包(com.sun.tools.attach)，方便开发者使用 Java 语言进行操作，这里我们使用其中提供的 loadAgent 方法实现运行中 agent 的能力。

```java
public class AttachUtil {

  public static void main(String[] args) throws IOException, AgentLoadException, AgentInitializationException, AttachNotSupportedException {

      // 获取运行中的JVM列表
      List<VirtualMachineDescriptor> vmList = VirtualMachine.list();
      // 需要agent的jar包路径
      String agentJar = "xxxx/agent-test.jar";
      for (VirtualMachineDescriptor vmd : vmList) {
          // 找到测试的JVM
          if (vmd.displayName().endsWith("WorkerMain")) {
              // attach到目标ID的JVM上
              VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
              // agent指定jar包到已经attach的JVM上
              virtualMachine.loadAgent(agentJar);
              virtualMachine.detach();
          }
      }
}
```

同时对之前启动时 Agent 的代码进行改写：

```java
public class AgentMain {

    // JVM启动时agent
    public static void premain(String args, Instrumentation inst) {
        agent0(args, inst);
    }

    // JVM运行时agent
    public static void agentmain(String args, Instrumentation inst) {
        agent0(args, inst);
    }

    public static void agent0(String args, Instrumentation inst) {
        System.out.println("agent is running!");
        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) {
                // 打印transform的类名
                System.out.println(className);
                return classfileBuffer;
            }
        },true);

        try {
            // 找到WorkerMain类，对其进行重定义
            Class<?> c = Class.forName("test.WorkerMain");
            inst.retransformClasses(c);
        } catch (Exception e) {
            System.out.println("error!");
        }
    }
}
```

这里我们也没有对字节码进行修改，还是直接返回原本的字节码。运行AttachUtil类，在目标JVM运行时完成了对其中test.WorkerMain 类的重新定义（虽然并没有修改字节码）。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221232010.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221232010" /></div>

### 流程解析

下面从JDK源码层面对整个流程进行浅析：

当 AttachUtil 的 loadAgent 方法调用时，目标 JVM 会调用自身的 Agent_OnAttach 方法，这个方法和之前提到的 Agent_OnLoad 方法类似，会进行 Agent JAR 包的解析，不同的是 Agent_OnAttach 方法会直接注册 ClassFileLoadHook 事件回调函数，然后执行 agentmain 方法添加类转换器。

需要注意的是我们在 Java 代码里调用了 Instrumentation#retransformClasses(Class<?>...) 方法，追踪代码可以发现最终调用了一个 native 方法，而这个 native 方法的实现则在 jdk 的 src\share\instrument\JPLISAgent.c 类中，最终 retransformClasses 会调用到 JVMTI 的 RetransformClasses 方法，这里由于JVM源码实现非常复杂，感兴趣的同学可以自行阅读(hotspot源码路径 src\share\vm\prims\jvmtiEnv.cpp)，简单来说在这个方法里，JVM 会触发 ClassFileLoadHook 事件回调完成类字节码的转换，并完成虚拟机内已经加载的类字节码的热替换。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221232136.png" style="display:block;margin-left:auto;margin-right:auto;width:70%;" alt="20220221232136" /></div>

至此，在JVM运行时悄无声息的完成了类的重定义，不得不佩服JDK开发者的高超手段。

## **运行方法分析**

了解到上述机制以后，我们可以通过在目标 JVM 运行时对其中的类进行重新定义，做到运行时插桩代码。

我们知道 ASM 是一个字节码修改框架，因此就可以在类转换器中，对原本类的字节码进行修改，然后再对这个类进行重定义(retransform)。

首先我们实现 ClassFileTransformer 接口，前文中在 transform 方法中并没有对于字节码进行修改，只是单纯的打印了一些信息，既然需要对字目标类的节码进行修改，我们需要了解下 ClassFileTransformer 接口中唯一需要实现的方法 transform，方法签名如下：  

```java
byte[] transform(ClassLoader         loader,
                 String              className,
                 Class<?>            classBeingRedefined,
                 ProtectionDomain    protectionDomain,
                 byte[]              classfileBuffer) throws IllegalClassFormatException;
```

可以看到方法入参有该类的类加载器、类名、类 Class 对象、类的保护域、以及最重要的 classfileBuffer，也就是这个类的字节码，此时就可以借助 ASM 这个字节码大杀器来为所欲为了。现在我们实现一个字节的类转换器 MyClassTransformer，然后使用 ASM 来对字节码进行修改。

```java
public class MyClassTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {

        // 对类字节码进行操作
        // 这里需要注意，不能对classfileBuffer这个数组进行修改操作
        try {
            // 创建ASM ClassReader对象，导入需要增强的对象字节码
            ClassReader reader = new ClassReader(classfileBuffer);
            ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
            
            // 自己实现的代码增强器
            MyEnhancer myEnhancer = new MyEnhancer(classWriter);

            // 增强字节码
            reader.accept(myEnhancer, ClassReader.SKIP_FRAMES);

            // 返回MyEnhancer增强后的字节码
            return classWriter.toByteArray();
        } catch (Exception e) {
            e.printStackTrace();
        }

        // return null 则不会对类进行转换
        return null;
    }
}
```

至此，我们拼上了 JVM 运行时插桩代码的最后一块拼图，这样就可以理解 Arthas 这类基于 Java Agent 的性能分析工具是如何在 JVM 运行时对你的代码进行了修改。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221232410.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221232410" /></div>

接着实现一个字节码增强器，借助 ASM 将对方法入参和方法耗时的监控代码织入，这里需要对字节码有一定了解，这里笔者使用到 ASM 提供的 AdviceAdapter 类简化开发。

```java
public class MyEnhancer extends ClassVisitor implements Opcodes {

    public MyEnhancer(ClassVisitor classVisitor) {
        super(ASM7, classVisitor);
    }

    /**
     * 对字节码中的方法定义进行修改
     */
    @Override
    public MethodVisitor visitMethod(int access, final String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
        if (isIgnore(mv, access, name)) {
            return mv;
        }
        return new AdviceAdapter(Opcodes.ASM7, new JSRInlinerAdapter(mv, access, name, descriptor, signature, exceptions), access, name, descriptor) {

            private final Type METHOD_CONTAINER = Type.getType(MethodContainer.class);
            private int timeIdentifier;
            private int argsIdentifier;

            /**
             * 进入方法前
             */
            @Override
            protected void onMethodEnter() {
                // 调用System.nanoTime()方法，将方法出参推入栈顶
                invokeStatic(Type.getType(System.class), Method.getMethod("long nanoTime()"));
                // 构造一个Long类型的局部变量，然后返回这个变量的标识符
                timeIdentifier = newLocal(Type.LONG_TYPE);

                // 存储栈顶元素也就是System.nanoTime()返回值，到指定位置本地变量区
                storeLocal(timeIdentifier);

                // 加载入参数组，将入参数组ref推入栈顶
                loadArgArray();
                // 构造一个Object[]类型的局部变量，返回这个变量的标识符
                argsIdentifier = newLocal(Type.getType(Object[].class));
                // 存储入参到指定位置本地变量区
                storeLocal(argsIdentifier);
            }

            @Override
            protected void onMethodExit(int opcode) {
                // 加载指定位置的本地变量到栈顶
                loadLocal(timeIdentifier);
                loadLocal(argsIdentifier);
                // 相当于调用MethodContainer.showMethod(long, Object[])方法
                invokeStatic(METHOD_CONTAINER, Method.getMethod("void showMethod(long,Object[])"));
            }
        };
    }


    /**
     * 方法是否需要被忽略（静态构造函数和构造函数）
     */
    private boolean isIgnore(MethodVisitor mv, int access, String methodName) {
        return null == mv
                || isAbstract(access)
                || isFinalMethod(access)
                || "<clinit>".equals(methodName)
                || "<init>".equals(methodName);
    }
  
    private boolean isAbstract(int access) {
        return (ACC_ABSTRACT & access) == ACC_ABSTRACT;
    }

    private boolean isFinalMethod(int methodAccess) {
        return (ACC_FINAL & methodAccess) == ACC_FINAL;
    }
}
```

由于这里对于字节码的修改是在方法内部，那么实现一些复杂逻辑的最好方式，就是调用外部类的静态方法，虚拟机字节码指令中的 invokestatic 是调用指定类的静态方法的指令，这里我们将方法开始时间和方法入参作为参数调用 MethodContainer.showMethod 方法，方法实现如下：

```java
public class MethodContainer {

    // 实现静态方法
    public static void showMethod(long startTime, Object[] Args) {
        System.out.println("方法耗时:" + (System.nanoTime() - startTime) / 1000000 + "ms, 方法入参：" + Arrays.toString(Args));
    }
}

// ASM操作字节码需要一定的学习才能理解，如果把上述字节码增强前后用Java代码表示大体入下：
// ASM代码增强前
public void test(int x) throws InterruptedException {
  	Thread.sleep(2000L);
    System.out.println("i'm working " + x);
}

// ASM代码增强后
public void test(int x) throws InterruptedException {
    long var2 = System.nanoTime();
    Object[] var4 = new Object[]{new Integer(x)};
    Thread.sleep(2000L);
    System.out.println("i'm working " + x);
    MethodContainer.showMethod(var2, var4);
}
```

最后运行 AttachUitl，可以看到正在运行中的 JVM 被成功的插入了我们实现的字节码，对于目标虚拟机来说是完全不需要任何实现的，而且被重定义的代码也可以被还原，感兴趣的同学可以自己了解下。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221232655.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220221232655" /></div>

对于 Java 开发者来说，代码插桩是很熟悉的一个概念，而且目前也有很多成熟的方式可以完成，比如说 Spring AOP实现采用的动态代理方式，Lombok 采用的插入式注解处理器方式等。

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220221232717.png" style="display:block;margin-left:auto;margin-right:auto;width:80%;" alt="20220221232717" /></div>

所谓术业有专攻，Instrument Agent 虽然强大，但也不见得适用所有的场景，对于日志统计、方法监控，动态代理已经能很好的满足这方面的需求，但是对于 JVM 性能监控或方法实时运行分析，Instrument Agent 可以随时插入、随时卸载、随时修改的特性就体现出了极大的优点，同时其基于 Java 代码开发又会相应的降低一些开发难度，这也是业内很多性能分析软件选择这种方式实现的原因。
