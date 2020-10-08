---
sort: 1
title: 自适应扩展
tags: Dubbo 自适应扩展 SPI
---

# 1. 原理

在 Dubbo 中，很多拓展都是通过 SPI 机制进行加载的，比如 Protocol、Cluster、LoadBalance 等。有时，有些拓展并**不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载**。这听起来有些矛盾。拓展未被加载，那么拓展方法就无法被调用（静态方法除外）。拓展方法未被调用，拓展就无法被加载。对于这个矛盾的问题，Dubbo 通过自适应拓展机制很好的解决了。自适应拓展机制的实现逻辑比较复杂，首先 Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类。最后再通过反射创建代理类，整个过程比较复杂。



_参考官方文档“造车轮”案例_



# 2. 源码分析

Adaptive 注解定义如下：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```

从上面的代码中可知，`Adaptive` 可注解在类或方法上。

- 当 `Adaptive` 注解在类上时，Dubbo 不会为该类生成代理类，该类型会缓存在`com.alibaba.dubbo.common.extension.ExtensionLoader#cachedAdaptiveClass`属性上，会直接实例化。

- 注解在方法（接口方法）上时，Dubbo 则会为该方法生成代理逻辑。

`Adaptive` 注解在类上的情况很少，在 Dubbo 中，仅有两个类被 Adaptive 注解了，分别是 `AdaptiveCompiler` 和 `AdaptiveExtensionFactory`。此种情况，表示拓展的加载逻辑由人工编码完成。

更多时候，Adaptive 是注解在接口方法上的，表示拓展的加载逻辑需由框架自动生成。注解在接口方法上时，处理逻辑较为复杂，本章将会重点分析此块逻辑。



## 2.1 获取自适应拓展

getAdaptiveExtension 方法是获取自适应拓展的入口方法，因此下面我们从这个方法进行分析。相关代码如下：

```java
// com.alibaba.dubbo.common.extension.ExtensionLoader#getAdaptiveExtension
public T getAdaptiveExtension() {
    // 从缓存中获取自适应拓展
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {    // 缓存未命中
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        // 创建自适应拓展
                        instance = createAdaptiveExtension();
                        // 设置自适应拓展到缓存中
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: ...");
                    }
                }
            }
        } else {
            throw new IllegalStateException("fail to create adaptive instance:  ...");
        }
    }

    return (T) instance;
}
```

getAdaptiveExtension 方法首先会检查缓存，缓存未命中，则调用 createAdaptiveExtension 方法创建自适应拓展。下面，我们看一下 createAdaptiveExtension 方法的代码。

```java
private T createAdaptiveExtension() {
    try {
        // 获取自适应拓展类，并通过反射实例化
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can not create adaptive extension ...");
    }
}
```

createAdaptiveExtension 包含了三个逻辑，分别如下：

1. 调用 getAdaptiveExtensionClass 方法获取自适应拓展 Class 对象
2. 通过反射进行实例化(无参构造)
3. 调用 injectExtension 方法向拓展实例中注入依赖

前两个逻辑比较好理解，第三个逻辑用于向自适应拓展对象中注入依赖。这个逻辑看似多余，但有存在的必要，这里简单说明一下。前面说过，Dubbo 中有两种类型的自适应拓展，一种是手工编码的，一种是自动生成的。**手工编码的自适应拓展中可能存在着一些依赖，而自动生成的 Adaptive 拓展则不会依赖其他类。这里调用 injectExtension 方法的目的是为手工编码的自适应拓展注入依赖**，[injectExtension 方法分析查看Dubbo SPI章节#Dubbo IOC](./dubbo-spi.md#3-2-Dubbo-IOC)。接下来，分析 getAdaptiveExtensionClass 方法的逻辑。

```
private Class<?> getAdaptiveExtensionClass() {
    // 通过 SPI 获取所有的拓展类,如果接口包含@SPI注解的name属性则缓存，用于查找默认实例
    getExtensionClasses();
    // 检查缓存，若缓存不为空，则直接返回缓存（实质上是某个扩展实现类上注解了@Adaptive，那么cachedAdaptiveClass就会存在）
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建自适应拓展类
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

`getAdaptiveExtensionClass` 方法同样包含了三个逻辑，如下：

1. 调用 `getExtensionClasses` 获取所有的拓展类

2. 检查缓存，若缓存不为空，则返回缓存

3. 若缓存为空，则调用 `createAdaptiveExtensionClass` 创建自适应拓展类

`getExtensionClasses` 这个方法用于获取某个接口的所有实现类。比如该方法可以获取 `Protocol` 接口的 `DubboProtocol`、`HttpProtocol`、`InjvmProtocol` 等实现类。在获取实现类的过程中，如果某个实现类被 `@Adaptive` 注解修饰了，那么该类就会被赋值给 `cachedAdaptiveClass` 变量。此时，上面步骤中的第二步条件成立（缓存不为空），直接返回 `cachedAdaptiveClass` 即可。如果所有的实现类均未被 `Adaptive` 注解修饰，那么执行第三步逻辑，创建自适应拓展类。相关代码如下：

```java
private Class<?> createAdaptiveExtensionClass() {
    // 构建自适应拓展代码
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    // 获取编译器实现类
    com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    // 编译代码，生成 Class
    return compiler.compile(code, classLoader);
}
```

`createAdaptiveExtensionClass` 方法用于生成自适应拓展类，该方法首先会生成自适应拓展类的源码，然后通过 `Compiler `实例（`Dubbo` 默认使用 `javassist` 作为编译器）编译源码，得到`代理类 Class 实例`。接下来，我们把重点放在代理类代码生成的逻辑上，其他逻辑大家自行分析。

## 2.2 自适应拓展类代码生成

### 2.2.1 Adaptive 注解检测

在生成代理类源码之前，createAdaptiveExtensionClassCode 方法首先会通过反射检测**接口方法**是否包含 `Adaptive` 注解。对于要生成自适应拓展的接口，**Dubbo 要求该接口至少有一个方法被 Adaptive 注解修饰**。若不满足此条件，就会抛出运行时异常。相关代码如下:

```java
private String createAdaptiveExtensionClassCode() {
        StringBuilder codeBuilder = new StringBuilder();
        Method[] methods = type.getMethods();
        boolean hasAdaptiveAnnotation = false;
        for (Method m : methods) {
            if (m.isAnnotationPresent(Adaptive.class)) {
                hasAdaptiveAnnotation = true;
                break;
            }
        }
        // no need to generate adaptive class since there's no adaptive method found.
        if (!hasAdaptiveAnnotation)
            throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");

        // 省略生成类...
}
```

### 2.2.2 生成类

通过 Adaptive 注解检测后，即可开始生成代码。代码生成的顺序与 Java 文件内容顺序一致，首先会生成 package 语句，然后生成 import 语句，紧接着生成类名等代码。整个逻辑如下：

```java
private String createAdaptiveExtensionClassCode() {
		// 省略 Adaptive 注解检测...

  	// 生成 package 代码：package + type 所在包
    codeBuilder.append("package ").append(type.getPackage().getName()).append(";");
    // 生成 import 代码：import + ExtensionLoader 全限定名
    codeBuilder.append("\nimport ").append(ExtensionLoader.class.getName()).append(";");
    // 生成类代码：public class + type简单名称 + $Adaptive + implements + type全限定名 + {
    codeBuilder.append("\npublic class ")
        .append(type.getSimpleName())
        .append("$Adaptive")
        .append(" implements ")
        .append(type.getCanonicalName())
        .append(" {");

    // ${生成方法}

    codeBuilder.append("\n}");
  	// 省略...
}
```

以 Dubbo 的 Protocol 接口为例，源码如下：

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```



生成的代码如下：

```java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
    // 省略方法代码
}
```

### 2.2.3 生成方法

一个方法可以被 Adaptive 注解修饰，也可以不被修饰。这里将未被 Adaptive 注解修饰的方法称为“无 Adaptive 注解方法”，下面我们先来看看此种方法的代码生成逻辑是怎样的。

#### 2.2.3.1 无 Adaptive 注解方法代码生成逻辑

对于接口方法，我们可以按照需求标注 Adaptive 注解。以 `Protocol`接口为例，该接口的 `destroy` 和 `getDefaultPort` 未标注 `Adaptive` 注解，其他方法均标注了 `Adaptive` 注解。Dubbo 不会为没有标注 `Adaptive` 注解的方法生成代理逻辑，对于该种类型的方法，仅会**生成一句抛出异常的代码**。生成逻辑如下：

```java
for (Method method : methods) {

    // 省略无关逻辑

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    // 如果方法上无 Adaptive 注解，则生成 throw new UnsupportedOperationException(...) 代码
    if (adaptiveAnnotation == null) {
        // 生成的代码格式如下：
        // throw new UnsupportedOperationException(
        //     "method " + 方法签名 + of interface + 全限定接口名 + is not adaptive method!”)
        code.append("throw new UnsupportedOperationException(\"method ")
            .append(method.toString()).append(" of interface ")
            .append(type.getName()).append(" is not adaptive method!\");");
    } else {
        // 省略无关逻辑
    }

    // 省略无关逻辑
}
```

以 Protocol 接口的 destroy 方法为例，上面代码生成的内容如下：

```java
    @Override
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }
```

#### 2.2.3.2 获取 URL 数据

前面说过方法代理逻辑会从 URL 中提取目标拓展的名称，因此代码生成逻辑的一个重要的任务是从方法的参数列表或者其他参数中获取 URL 数据（com.alibaba.dubbo.common.URL）。举例说明一下，我们要为 Protocol 接口的 refer 和 export 方法生成代理逻辑。在运行时，通过反射得到的方法定义大致如下：

```java
Invoker refer(Class<T> arg0, URL arg1) throws RpcException;
Exporter export(Invoker<T> arg0) throws RpcException;
```

- 对于 refer 方法，通过遍历 refer 的参数列表即可获取 URL 数据。

- 对于 export 方法，export 参数列表中没有 URL 参数，因此需要从 Invoker 参数中获取 URL 数据。**获取方式是调用 Invoker 中可返回 URL 的 getter 方法**，比如 getUrl。如果 Invoker 中无相关 getter 方法，此时则会抛出异常。整个逻辑如下：

```java
for (Method method : methods) {
  	// 方法返回值类型
    Class<?> rt = method.getReturnType();
  	// 方法参数类型数组
    Class<?>[] pts = method.getParameterTypes();
  	// 方法异常类型数组
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成逻辑}
    } else {
    	  int urlTypeIndex = -1;
        // 遍历参数列表，确定 URL 参数位置
        for (int i = 0; i < pts.length; ++i) {
            if (pts[i].equals(URL.class)) {
                urlTypeIndex = i;
                break;
            }
        }

        // urlTypeIndex != -1，表示参数列表中存在 URL 参数
        if (urlTypeIndex != -1) {
            // 为 URL 类型参数生成判空代码，格式如下：
            // if (arg + urlTypeIndex == null)
            //     throw new IllegalArgumentException("url == null");
          	// 例如：
          	// if (arg1 == null)
            //     throw new IllegalArgumentException("url == null");
            String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
                                     urlTypeIndex);
            code.append(s);

            // 为 URL 类型参数生成赋值代码，形如 URL url = arg1
            s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
            code.append(s);


        // 参数列表中不存在 URL 类型参数
        } else {
            String attribMethod = null;

            LBL_PTS:
            // 遍历方法的参数类型列表
            for (int i = 0; i < pts.length; ++i) {
                // 获取某一类型参数的全部方法
                Method[] ms = pts[i].getMethods();
                // 遍历方法列表，寻找可返回 URL 的 getter 方法
                for (Method m : ms) {
                    String name = m.getName();
                    // 1. 方法名以 get 开头，或方法名大于3个字符
                    // 2. 方法的访问权限为 public
                    // 3. 非静态方法
                    // 4. 方法参数数量为0
                    // 5. 方法返回值类型为 URL
                    if ((name.startsWith("get") || name.length() > 3)
                        && Modifier.isPublic(m.getModifiers())
                        && !Modifier.isStatic(m.getModifiers())
                        && m.getParameterTypes().length == 0
                        && m.getReturnType() == URL.class) {
                        urlTypeIndex = i;
                        attribMethod = name;

                        // 结束 for (int i = 0; i < pts.length; ++i) 外层循环
                        break LBL_PTS;
                    }
                }
            }
            if (attribMethod == null) {
                // 如果所有参数中均不包含可返回 URL 的 getter 方法，则抛出异常
                throw new IllegalStateException("fail to create adaptive class for interface ...");
            }

            // 为可返回 URL 的参数生成判空代码，格式如下：
            // if (arg + urlTypeIndex == null)
            //     throw new IllegalArgumentException("参数全限定名 + argument == null");
            String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
                                     urlTypeIndex, pts[urlTypeIndex].getName());
            code.append(s);

            // 为 getter 方法返回的 URL 生成判空代码，格式如下：
            // if (argN.getter方法名() == null)
            //     throw new IllegalArgumentException(参数全限定名 + argument getUrl() == null);
            s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
                              urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
            code.append(s);

            // 生成赋值语句，格式如下：
            // URL全限定名 url = argN.getter方法名()，比如
            // com.alibaba.dubbo.common.URL url = invoker.getUrl();
            s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
            code.append(s);
        }

        // 省略无关代码
    }

    // 省略无关代码
}
```

简化各分支含义：

- 方法没Adaptive注解

  添加方法直接抛异常代码

- 方法有Adaptive注解

  - 方法参数里有com.alibaba.dubbo.common.URL类型参数的

    添加判空并设置url属性代码

  - 方法参数里没有URL类型参数的

    逐个查找每个参数的方法，寻找URL的getter方法，有就添加判空和设置url属性代码，没有则报错

这段代码**主要目的是为了获取 URL 数据，并为之生成判空和赋值代码**。以 Protocol 的 refer 和 export 方法为例，上面的代码为它们生成如下内容（代码已格式化）：

```java
refer:
if (arg1 == null)
    throw new IllegalArgumentException("url == null");
com.alibaba.dubbo.common.URL url = arg1;

export:
if (arg0 == null)
    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null)
    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
com.alibaba.dubbo.common.URL url = arg0.getUrl();
```

#### 2.2.3.3 获取 Adaptive 注解值

Adaptive 注解值 value 类型为 String[]，可填写多个值，默认情况下为空数组。若 value 为非空数组，直接获取数组内容即可。若 value 为空数组，则需进行额外处理。处理过程是将类名转换为字符数组，然后遍历字符数组，并将字符放入 StringBuilder 中。若字符为大写字母，则向 StringBuilder 中添加点号，随后将字符变为小写存入 StringBuilder 中。比如 LoadBalance 经过处理后，得到 load.balance。

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成逻辑}
    } else {
        // ${获取 URL 数据}

      	// 获取方法上的@Adaptive注解的value属性
        String[] value = adaptiveAnnotation.value();
        // value 为空数组
        if (value.length == 0) {
            // 获取类名，并将类名转换为字符数组
            char[] charArray = type.getSimpleName().toCharArray();
            StringBuilder sb = new StringBuilder(128);
            // 遍历字节数组
            for (int i = 0; i < charArray.length; i++) {
                // 检测当前字符是否为大写字母
                if (Character.isUpperCase(charArray[i])) {
                    if (i != 0) {
                        // 向 sb 中添加点号
                        sb.append(".");
                    }
                    // 将字符变为小写，并添加到 sb 中
                    sb.append(Character.toLowerCase(charArray[i]));
                } else {
                    // 添加字符到 sb 中
                    sb.append(charArray[i]);
                }
            }
            value = new String[]{sb.toString()};
        }

        // 省略无关代码
    }

    // 省略无关逻辑
}
```

#### 2.2.3.4 检测 Invocation 参数

此段逻辑是检测方法列表中是否存在 Invocation 类型的参数，若存在，则为其生成判空代码和其他一些代码。相应的逻辑如下：

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();    // 获取参数类型列表
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // ${无 Adaptive 注解方法代码生成逻辑}
    } else {
        // ${获取 URL 数据}

        // ${获取 Adaptive 注解值}

        boolean hasInvocation = false;
        // 遍历参数类型列表
        for (int i = 0; i < pts.length; ++i) {
            // 判断当前参数名称是否等于 com.alibaba.dubbo.rpc.Invocation
            if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
                // 为 Invocation 类型参数生成判空代码
                String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
                code.append(s);
                // 生成 getMethodName 方法调用代码，格式为：
                //    String methodName = argN.getMethodName();
                s = String.format("\nString methodName = arg%d.getMethodName();", i);
                code.append(s);

                // 设置 hasInvocation 为 true
                hasInvocation = true;
                break;
            }
        }
    }

    // 省略无关逻辑
}
```

#### 2.2.3.5 生成拓展名获取逻辑

本段逻辑用于根据 SPI 和 Adaptive 注解值生成“获取拓展名逻辑”，同时生成逻辑也受 Invocation 类型参数影响，综合因素导致本段逻辑相对复杂。本段逻辑可能会生成但不限于下面的代码：

```java
String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
```

或

```java
String extName = url.getMethodParameter(methodName, "loadbalance", "random");
```

亦或是

```java
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
```

本段逻辑复杂之处在于条件分支比较多，大家在阅读源码时需要知道每个条件分支的意义是什么，否则不太容易看懂相关代码。下面开始分析本段逻辑。

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成逻辑}
    } else {
        // ${获取 URL 数据 或报错}

        // ${获取 Adaptive 注解值 或使用类名转化AbCd->ab.cd值}

        // ${检测 Invocation 参数}

        // 设置默认拓展名，cachedDefaultName 源于 SPI 注解值，默认情况下，
        // SPI 注解值为空串，此时 cachedDefaultName = null
        String defaultExtName = cachedDefaultName;
        String getNameCode = null;

        // 遍历 value，这里的 value 是 Adaptive 的注解值，2.2.3.3 节分析过 value 变量的获取过程。
        // 此处循环目的是生成从 URL 中获取拓展名的代码，生成的代码会赋值给 getNameCode 变量。注意这
        // 个循环的遍历顺序是由后向前遍历的。
        for (int i = value.length - 1; i >= 0; --i) {
            // 当 i 为最后一个元素的坐标时（第一次进入循环）
            if (i == value.length - 1) {
                // 默认拓展名非空(@SPI存在value属性)
                if (null != defaultExtName) {
                    // protocol 是 url 的一部分，可通过 getProtocol 方法获取，其他的则是从
                    // URL 参数中获取。因为获取方式不同，所以这里要判断 value[i] 是否为 protocol
                    if (!"protocol".equals(value[i]))
                    	// hasInvocation 用于标识方法参数列表中是否有 Invocation 类型参数
                        if (hasInvocation)
                            // 生成的代码功能等价于下面的代码：
                            //   url.getMethodParameter(methodName, value[i], defaultExtName)
                            // 以 LoadBalance 接口的 select 方法为例，最终生成的代码如下：
                            //   url.getMethodParameter(methodName, "loadbalance", "random")
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                    	else
                    		// 生成的代码功能等价于下面的代码：（查询）
	                        //   url.getParameter(value[i], defaultExtName)
	                        getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                    else
                    	// 生成的代码功能等价于下面的代码：
                        //   ( url.getProtocol() == null ? defaultExtName : url.getProtocol() )
                        getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);

                // 默认拓展名为空
                } else {
                    if (!"protocol".equals(value[i]))
                        if (hasInvocation)
                        	// 生成代码格式同上
                            getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
	                    else
	                    	// 生成的代码功能等价于下面的代码：
	                        //   url.getParameter(value[i])
	                        getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                    else
                    	// 生成从 url 中获取协议的代码，比如 "dubbo"
                        getNameCode = "url.getProtocol()";
                }
            } else {
                if (!"protocol".equals(value[i]))
                    if (hasInvocation)
                        // 生成代码格式同上
                        getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
	                else
	                	// 生成的代码功能等价于下面的代码：
	                    //   url.getParameter(value[i], getNameCode)
	                    // 以 Transporter 接口的 connect 方法为例，最终生成的代码如下：
	                    //   url.getParameter("client", url.getParameter("transporter", "netty"))
	                    getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                else
                    // 生成的代码功能等价于下面的代码：
                    //   url.getProtocol() == null ? getNameCode : url.getProtocol()
                    // 以 Protocol 接口的 connect 方法为例，最终生成的代码如下：
                    //   url.getProtocol() == null ? "dubbo" : url.getProtocol()
                    getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
            }
        }
        // 生成 extName 赋值代码
        code.append("\nString extName = ").append(getNameCode).append(";");
        // 生成 extName 判空代码
        String s = String.format("\nif(extName == null) " +
                                 "throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
                                 type.getName(), Arrays.toString(value));
        code.append(s);
    }

    // 省略无关逻辑
}
```

上面代码比较复杂，不是很好理解。对于这段代码，建议大家写点测试用例，对 Protocol、LoadBalance 以及 Transporter 等接口的自适应拓展类代码生成过程进行调试。这里我以 Transporter 接口的自适应拓展类代码生成过程举例说明。首先看一下 Transporter 接口的定义，如下：

```java
@SPI("netty")
public interface Transporter {
	// @Adaptive({server, transporter})
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;

    // @Adaptive({client, transporter})
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```

下面对 connect 方法代理逻辑生成的过程进行分析，此时生成代理逻辑所用到的变量如下：

```properties
String defaultExtName = "netty";
boolean hasInvocation = false;
String getNameCode = null;
String[] value = ["client", "transporter"];
```

下面对 value 数组进行遍历，此时 i = 1, value[i] = "transporter"，生成的代码如下：

```java
getNameCode = url.getParameter("transporter", "netty");
```

接下来，for 循环继续执行，此时 i = 0, value[i] = "client"，生成的代码如下：

```java
getNameCode = url.getParameter("client", url.getParameter("transporter", "netty"));
```

for 循环结束运行，现在为 extName 变量生成赋值和判空代码，如下：

```java
String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
if (extName == null) {
    throw new IllegalStateException(
        "Fail to get extension(com.alibaba.dubbo.remoting.Transporter) name from url(" + url.toString()
        + ") use keys([client, transporter])");
}
```

实质上查询URL的参数时，根据方法注解值`@Adaptive({client, transporter})`或者没有注解属性值时使用接口名称的格式转换`AbCd->ab.cd`值，优先级是左边最高，没有才向右查询，通过`url.getParameter`获取，都没有才使用`@SPI`注解的值作为默认值。当为属性值是`protocol`时比较特殊，是通过`url.getProtocol()`获取。另外方法里有`Invocation`类型参数的用`url.getMethodParameter(methodName,key,defaultExtName)`获取。

#### 2.2.3.6 生成拓展加载与目标方法调用逻辑

本段代码逻辑用于根据拓展名加载拓展实例，并调用拓展实例的目标方法。相关逻辑如下：

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成逻辑}
    } else {
        // ${获取 URL 数据}

        // ${获取 Adaptive 注解值}

        // ${检测 Invocation 参数}

        // ${生成拓展名获取逻辑}

        // 生成拓展获取代码，格式如下：
        // type全限定名 extension = (type全限定名)ExtensionLoader全限定名
        //     .getExtensionLoader(type全限定名.class).getExtension(extName);
        // Tips: 格式化字符串中的 %<s 表示使用前一个转换符所描述的参数，即 type 全限定名
        s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
                        type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
        code.append(s);

		// 如果方法返回值类型非 void，则生成 return 语句。
        if (!rt.equals(void.class)) {
            code.append("\nreturn ");
        }

        // 生成目标方法调用逻辑，格式为：
        //     extension.方法名(arg0, arg2, ..., argN);
        s = String.format("extension.%s(", method.getName());
        code.append(s);
        for (int i = 0; i < pts.length; i++) {
            if (i != 0)
                code.append(", ");
            code.append("arg").append(i);
        }
        code.append(");");
    }

    // 省略无关逻辑
}
```

以 Protocol 接口举例说明，上面代码生成的内容如下：

```java
com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
    .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1);
```

#### 2.2.3.7 生成完整的方法

本节进行代码生成的收尾工作，主要用于生成方法定义的代码。相关逻辑如下：

```java
for (Method method : methods) {
    Class<?> rt = method.getReturnType();
    Class<?>[] pts = method.getParameterTypes();
    Class<?>[] ets = method.getExceptionTypes();

    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        // $无 Adaptive 注解方法代码生成逻辑}
    } else {
        // ${获取 URL 数据}

        // ${获取 Adaptive 注解值}

        // ${检测 Invocation 参数}

        // ${生成拓展名获取逻辑}

        // ${生成拓展加载与目标方法调用逻辑}

        // public + 返回值全限定名 + 方法名 + (
        codeBuilder.append("\npublic ")
            .append(rt.getCanonicalName())
            .append(" ")
            .append(method.getName())
            .append("(");

        // 添加参数列表代码
        for (int i = 0; i < pts.length; i++) {
            if (i > 0) {
                codeBuilder.append(", ");
            }
            codeBuilder.append(pts[i].getCanonicalName());
            codeBuilder.append(" ");
            codeBuilder.append("arg").append(i);
        }
        codeBuilder.append(")");

        // 添加异常抛出代码
        if (ets.length > 0) {
            codeBuilder.append(" throws ");
            for (int i = 0; i < ets.length; i++) {
                if (i > 0) {
                    codeBuilder.append(", ");
                }
                codeBuilder.append(ets[i].getCanonicalName());
            }
        }
        codeBuilder.append(" {");
        codeBuilder.append(code.toString());
        codeBuilder.append("\n}");
    }
}

```

以 Protocol 的 refer 方法为例，上面代码生成的内容如下：

```java
public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) {
    // 方法体
}
```



# 总结

> SPI自适应扩展，是通过**自适应扩展实现类**来统一管理当前接口的其他扩展实现类的方式，**实现运行时动态获取扩展实现类并调用方法**。

其中`@Adaptive`注解在实现类上的目前只有`AdaptiveCompiler`和`AdaptiveExtensionFactory`，表示直接使用当前实现类作为**自适应扩展实现类**；更多的是注解在**接口方法**上，表示拓展实现类需由框架自动生成，生成的类基本功能是大致如下：

	1. 获取扩展名。获取接口方法上`@Adaptive`注解属性值作为key，查询`URL`对应key的属性值，如果没有则使用接口上`@SPI`注解属性值作为默认扩展名；
 	2. 通过扩展名查找扩展实现，并调用同名方法。



框架生成自适应实现类代码的流程大致包括：

1. 校验接口方法，至少有一个方法上有`@Adaptive`注解

2. 生成类语句

   1. 包名、依赖、类名代码，类名规则为接口简单名字+$Adaptive；

   2. 方法实现(逐个处理)：

      - 没有` @Adaptive`注解的方法实现是返回抛异常代码（也就是说业务上不应该走到这个方法）

      - 有` @Adaptive`注解的方法（重要🌟，通过URL获取指定的配置值，用于动态获取指定扩展实现类）

        1. 获取`com.alibaba.dubbo.common.URL`参数。（ `URL` 作为配置信息的统一格式，所有扩展点都通过传递 URL 携带配置信息。）
           - 方法参数中直接存在`URL`，那么记录参数位置
           - 不存在的，则遍历所有参数对象，查找返回`URL`的`getter`方法，有的话则记录参数位置和方法名，没有报错；

        2. 获取配置名称列表。

           - `@Adaptive`注解存在`value`，则直接使用;
           - 不存在`value`，则使用接口名称的简单名称格式转化`AbCd->ab.cd`;

        3. 处理`Invocation`参数。如果方法存在该类型参数的，生成相应的判空和获取方法名称`argN.getMethodName()`代码

        4. 生成获取扩展名获取代码。

           - 代码功能：通过`URL`获取**配置名称**的**值**，配置名称列表最左边的优先级最高，如果全部都获取不到，则使用`@SPI`注解的`value`属性值(默认值)

           - URL获取配置的不同条件下的实现方式：
             1. 如果配置名称是`protocol`，则生成例如`String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());`
             2. 如果不是，则
                1. 如果参数存在`Invocation`类型参数，则生成例如：`String extName = url.getMethodParameter(methodName, "loadbalance", "random");`
                2. 其他则生成例如：`String extName = url.getParameter("client", url.getParameter("transporter", "netty"));`

        5. 生成拓展加载与目标方法调用逻辑。

           - 代码功能：通过上一步获取的唯一扩展名，通过SPI方式获取扩展实现类，然后调用同名方法。

        6. 生成完整的方法。方法名、返回类型、参数等代码补充。



# 彩蛋

## Protocol自适应扩展类反编译

使用`arthas`反编译命令`jad com.alibaba.dubbo.rpc.Protocol$Adaptive`

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.alibaba.dubbo.rpc.Exporter;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.Protocol;
import com.alibaba.dubbo.rpc.RpcException;

public class Protocol$Adaptive
implements Protocol {
    @Override
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    @Override
    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public Exporter export(Invoker invoker) throws RpcException {
        String string;
        if (invoker == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(").append(uRL.toString()).append(") use keys([protocol])").toString());
        }
        Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.export(invoker);
    }

    public Invoker refer(Class class_, URL uRL) throws RpcException {
        String string;
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string2 = string = uRL2.getProtocol() == null ? "dubbo" : uRL2.getProtocol();
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(").append(uRL2.toString()).append(") use keys([protocol])").toString());
        }
        Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
        return protocol.refer(class_, uRL);
    }
}

```



## ProxyFactory自适应扩展类反编译

使用`arthas`反编译命令`jad com.alibaba.dubbo.rpc.ProxyFactory$Adaptive`


```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.URL;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.alibaba.dubbo.rpc.Invoker;
import com.alibaba.dubbo.rpc.ProxyFactory;
import com.alibaba.dubbo.rpc.RpcException;

public class ProxyFactory$Adaptive
implements ProxyFactory {
    public Invoker getInvoker(Object object, Class class_, URL uRL) throws RpcException {
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string = uRL2.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(").append(uRL2.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getInvoker(object, class_, uRL);
    }

    public Object getProxy(Invoker invoker, boolean bl) throws RpcException {
        if (invoker == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string = uRL.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(").append(uRL.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getProxy(invoker, bl);
    }

    public Object getProxy(Invoker invoker) throws RpcException {
        if (invoker == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string = uRL.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(").append(uRL.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getProxy(invoker);
    }
}

```

