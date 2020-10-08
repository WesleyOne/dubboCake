---
sort: 1
title: Dubbo SPI
tags: Dubbo SPI
---


## 1. Java SPI示例

`SPI` 全称为 `Service Provider Interface`，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。正因此特性，我们可以很容易的通过 SPI 机制为我们的程序提供拓展功能。

JAVA SPI 使用案例如下：

- 接口

  ```java
  package com.example.provider;

  public interface Robot {
      void sayHello();
  }
  ```

- 扩展实现

  ```java
  package com.example.provider;

  public class Bumblebee implements Robot {

      @Override
      public void sayHello() {
          System.out.println("Hello, I am com.example.provider.Bumblebee.");
      }
  }
  ```

  ```java
  package com.example.provider;

  public class OptimusPrime implements Robot {

      @Override
      public void sayHello() {
          System.out.println("Hello, I am Optimus Prime.");
      }
  }
  ```

- 配置文件

  - 在资源目录下创建`META-INF/services`目录；

  - 然后在该目录下创建以接口全限定名命名的配置文件，譬如`com.example.provider.Robot`；

  - 然后在该配置文件内添加实现类的全限定名。

    配置文件如下图所示：

    ![Java SPI 配置文件]({{ site.baseurl }}/assets/images/dubbo_spi/java_spi.png){:.border.rounded}

- 加载并执行扩展实现（测试用例如下）

  ```java
  import com.example.provider.Robot;
  import org.junit.Test;

  import java.util.ServiceLoader;

  public class JavaSPITest {

      @Test
      public void sayHello() throws Exception {
          ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
          System.out.println("Java SPI");
          serviceLoader.forEach(Robot::sayHello);

          // 输出内容：
          // Java SPI
  				// Hello, I am Optimus Prime.
  				// Hello, I am com.example.provider.Bumblebee.
      }
  }
  ```



## 2. Dubbo SPI示例

Dubbo 并未使用 Java 原生的 SPI 机制，**与 Java SPI 实现类配置不同，Dubbo SPI 是通过键值对的方式进行配置，这样我们可以按需加载指定的实现类。**



- 接口、扩展实现使用Java SPI相同的类，**但是需要在 Robot 接口上标注 @SPI 注解**

  ```java
  package com.example.provider;

  import com.alibaba.dubbo.common.extension.SPI;

  @SPI
  public interface Robot {
      void sayHello();
  }
  ```

- 配置文件

  - 在资源目录下创建`META-INF/dubbo`目录；
  - 然后在该目录下创建以接口全限定名命名的配置文件，譬如`com.example.provider.Robot`；
  - 然后在该配置文件内添加键值对(key=value)，key是自定义的，value是实现类的全限定名。

  配置文件如下图所示：

  ![Dubbo SPI示例配置]({{ site.baseurl }}/assets/images/dubbo_spi/dubbo_spi.png){:.border.rounded}

- 加载并执行扩展实现（测试用例如下）

```java
import com.alibaba.dubbo.common.extension.ExtensionLoader;
import com.example.provider.Robot;
import org.junit.Test;

public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

## 3. Dubbo SPI源码分析

根据上文的扩展实现，首先通过 `ExtensionLoader` 的 `getExtensionLoader` 方法获取一个 `ExtensionLoader` 实例，然后再通过 `ExtensionLoader` 的 `getExtension` 方法获取拓展类对象。这其中，`getExtensionLoader` 方法用于从缓存中获取与拓展类对应的 `ExtensionLoader`，若缓存未命中，则创建一个新的实例。（其实`getExtensionLoader`里面也有通过SPI机制加载了`ExtensionFactory`的过程）

下面我们从 `ExtensionLoader` 的 `getExtension` 方法作为入口，对拓展类对象的获取过程进行详细的分析。

```java
		// com.alibaba.dubbo.common.extension.ExtensionLoader#getExtension
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
          	// 获取默认的拓展实现类
            return getDefaultExtension();
        }
      	// Holder，顾名思义，用于持有目标对象
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
	      // 双重检查
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                  	// 创建拓展实例
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

首先检查缓存，缓存未命中则创建拓展对象。下面介绍创建拓展实例`createExtension`方法。

```java
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“扩展名称”到“扩展类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例（无参构造）
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例,比如Protocal就会被ProtocolFilterWrapper等Wapper包裹
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

createExtension 方法的逻辑稍复杂一下，包含了如下的步骤：

1. 通过 getExtensionClasses 获取所有的拓展类
2. 通过反射创建拓展对象
3. 向拓展对象中注入依赖
4. 将拓展对象包裹在相应的 Wrapper 对象中

以上步骤中，第一个步骤是加载拓展类的关键，第三和第四个步骤是 Dubbo IOC 与 AOP 的具体实现。在接下来的章节中，将会重点分析 getExtensionClasses 方法的逻辑，以及简单介绍 Dubbo IOC 的具体实现。

### 3.1 获取所有的拓展类
我们在通过名称获取拓展类之前，首先需要根据配置文件解析出拓展项名称到拓展类的映射关系表（Map<名称, 拓展类>），之后再根据拓展项名称从映射关系表中取出相应的拓展类即可。相关过程的代码分析如下：

```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

这里也是先检查缓存，若缓存未命中，则通过 synchronized 加锁。加锁后再次检查缓存，并判空。此时如果 classes 仍为 null，则通过 loadExtensionClasses 加载拓展类。下面分析 loadExtensionClasses 方法的逻辑。

```
private Map<String, Class<?>> loadExtensionClasses() {
    // 获取 SPI 注解，这里的 type 变量是在调用 getExtensionLoader 方法时传入的，（实质上就是SPI接口类！）
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            // 对 SPI 注解内容进行切分
            String[] names = NAME_SEPARATOR.split(value);
            // 检测 SPI 注解内容是否合法，不合法则抛出异常
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension...");
            }

            // 设置默认名称，参考 getDefaultExtension 方法
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    // 加载指定文件夹下的配置文件
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);// META-INF/dubbo/internal/
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);// META-INF/dubbo/
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);// META-INF/services/
    return extensionClasses;
}
```

loadExtensionClasses 方法总共做了两件事情，一是对 SPI 注解进行解析，二是调用 loadDirectory 方法加载指定文件夹配置文件。SPI 注解解析就是获取下注解的`value`属性值，这个值就是默认使用的实现的映射key。下面我们来看一下 loadDirectory 做了哪些事情。

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    // fileName = 文件夹路径 + type 全限定名 ,例如：META-INF/dubbo/internal/com.example.provider.Robot
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        // 根据文件名加载所有的同名文件
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 加载资源
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("...");
    }
}
```

loadDirectory 方法先通过 classLoader 获取所有资源链接，然后再通过 loadResource 方法加载资源。我们继续跟下去，看一下 loadResource 方法的实现。

```java
private void loadResource(Map<String, Class<?>> extensionClasses,
	ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(resourceURL.openStream(), "utf-8"));
        try {
            String line;
            // 按行读取配置内容
            while ((line = reader.readLine()) != null) {
                // 定位 # 字符
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    // 截取 # 之前的字符串，# 之后的内容为注释，需要忽略
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            // 以等于号 = 为界，截取键与值
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                            // 加载类，并通过 loadClass 方法对类进行缓存
                            loadClass(extensionClasses, resourceURL,
                                      Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class...");
                    }
                }
            }
        } finally {
            reader.close();
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class...");
    }
}
```

以下格式均支持：

```properties
key0=value0
key1=value1#注释
key2 = value2 #注释
```

loadResource 方法用于读取和解析配置文件，并通过反射加载类，最后调用 loadClass 方法进行其他操作。loadClass 方法用于主要用于操作缓存，该方法的逻辑如下：

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL,
    Class<?> clazz, String name) throws NoSuchMethodException {
    // 父类.class.isAssignableFrom(子类.class)
    // 判断待加载的类是否实现对应接口
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("...");
    }

    // 检测目标类上是否有 Adaptive 注解
    // 当调用getAdaptiveExtensionClass时，会优先使用该注解的实现，比如ExtensionFactory的AdaptiveExtensionFactory
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            // 设置 cachedAdaptiveClass缓存
            cachedAdaptiveClass = clazz;
          	// 同一个接口不能有多个Adaptive注解的实现
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("...");
        }

    // 检测 clazz 是否是 Wrapper 类型
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        // 存储 clazz 到 cachedWrapperClasses 缓存中
        wrappers.add(clazz);

    // 程序进入此分支，表明 clazz 是一个普通的拓展类
    } else {
        // 检测 clazz 是否有默认的构造方法，如果没有，则抛出异常
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            // 如果 name 为空，则尝试从 Extension 注解中获取 name，或使用小写的类名作为 name
            // 2.6.4版本Extension注解以过期，应该是兼容老版本使用
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("...");
            }
        }
        // 切分 name
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                // 如果类上有 Activate 注解，则使用 names 数组的第一个元素作为键，
                // 存储 name 到 Activate 注解对象的映射关系
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    // 存储 Class 到名称的映射关系
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    // 存储名称到 Class 的映射关系
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("...");
                }
            }
        }
    }
}

// 判断包装类：带有接口类型参数的构造函数
private boolean isWrapperClass(Class<?> clazz) {
  try {
    clazz.getConstructor(type);
    return true;
  } catch (NoSuchMethodException e) {
    return false;
  }
}
```

插入知识点`isAssignableFrom`和`instanceof`区别：

```
父类.class.isAssignableFrom(子类.class)
子类实例 instanceof 父类.class
```

如上，loadClass 方法操作了不同的缓存，比如 cachedAdaptiveClass、cachedWrapperClasses 和 cachedNames 等等。到此，关于缓存类加载的过程就分析完了。

### 3.2 Dubbo IOC

Dubbo IOC 是通过 setter 方法注入依赖。Dubbo 首先会通过反射获取到实例的所有方法，然后再遍历方法列表，检测方法名是否具有 setter 方法特征。若有，则通过 ObjectFactory 获取依赖对象，最后通过反射调用 setter 方法将依赖设置到目标对象中。整个过程对应的代码如下：

```
// 在如下方法前，上一步获取的class后，通过clazz.newInstance()无参构造实例化。
// com.alibaba.dubbo.common.extension.ExtensionLoader#injectExtension
private T injectExtension(T instance) {
    try {
        if (objectFactory != null) {
            // 遍历目标类的所有方法
            for (Method method : instance.getClass().getMethods()) {
                // 检测方法是否以 set 开头，且方法仅有一个参数，且方法访问级别为 public
                if (method.getName().startsWith("set")
                    && method.getParameterTypes().length == 1
                    && Modifier.isPublic(method.getModifiers())) {
                    // 获取 setter 方法参数类型
                    Class<?> pt = method.getParameterTypes()[0];
                    try {
                        // 获取属性名，比如 setName 方法对应属性名 name
                        String property = method.getName().length() > 3 ?
                            method.getName().substring(3, 4).toLowerCase() +
                            	method.getName().substring(4) : "";
                        // 从 ObjectFactory 中获取依赖对象
                        Object object = objectFactory.getExtension(pt, property);
                        if (object != null) {
                            // 通过反射调用 setter 方法设置依赖
                            method.invoke(instance, object);
                        }
                    } catch (Exception e) {
                        logger.error("fail to inject via method...");
                    }
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

在上面代码中，`objectFactory`是在`ExtensionLoader.getExtensionLoader(type);`下的`new ExtensionLoader<T>(type)`时，通过SPI方式获取，`ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()`方法获取到`AdaptiveExtensionFactory`实现，`AdaptiveExtensionFactory` 内部维护了一个 `ExtensionFactory`列表，用于存储其他类型的 `ExtensionFactory`。`Dubbo` 目前提供了两种 `ExtensionFactory`，分别是 `SpiExtensionFactory` 和 `SpringExtensionFactory`。前者用于创建自适应的拓展，后者是用于从 Spring 的 IOC 容器中获取所需的拓展。

找到`META-INF/dubbo/internal`下的`com.alibaba.dubbo.common.extension.ExtensionFactory`文件

```properties
adaptive=com.alibaba.dubbo.common.extension.factory.AdaptiveExtensionFactory
spi=com.alibaba.dubbo.common.extension.factory.SpiExtensionFactory
spring=com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory
# 省略。不知道为啥重复写了两遍
```

> @Adaptive 注解的扩展只保存在ExtensionLoader.cachedAdaptiveClass,不会在扩展集合缓存cachedNames中

```java
/**
 * AdaptiveExtensionFactory
 *
 */
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

		// 维护的扩展实例集合
    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
    		// 将扩展集合里的所有扩展类实例化（@Adaptive注解的扩展不在扩展集合中，不会出现死循环）
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
    		// 逐个扩展找对应实例
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

```java
/**
 * SpiExtensionFactory
 * 根据给的类型获取SPI机制下的@Adaptive实例或者通过javassist动态字节码技术生成实例
 */
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            if (!loader.getSupportedExtensions().isEmpty()) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }

}
```

```java
package com.alibaba.dubbo.config.spring.extension;
// 省略...

/**
 * SpringExtensionFactory
 */
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();

    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }

    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    // currently for test purpose
    public static void clearContexts() {
        contexts.clear();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {
      	// 从Spring上下文中通过别名查找实例
        for (ApplicationContext context : contexts) {
            if (context.containsBean(name)) {
                Object bean = context.getBean(name);
              	// 判断实例是否是对应的类型
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }

        logger.warn("No spring extension(bean) named:" + name + ", try to find an extension(bean) of type " + type.getName());

      	// 上面找不到，再根据类型查找实例
        for (ApplicationContext context : contexts) {
            try {
                return context.getBean(type);
            } catch (NoUniqueBeanDefinitionException multiBeanExe) {
                throw multiBeanExe;
            } catch (NoSuchBeanDefinitionException noBeanExe) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Error when get spring extension(bean) for type:" + type.getName(), noBeanExe);
                }
            }
        }

        logger.warn("No spring extension(bean) named:" + name + ", type:" + type.getName() + " found, stop get bean.");

        return null;
    }

}

```



## 总结

本章节介绍了Java SPI和Dubbo SPI的使用，以及Dubbo SPI 的加载拓展类的流程。流程简化如下：

1. 通过接口全限定名称查找指定路径下的同名文件

   路径包括：META-INF/dubbo/internal/、META-INF/dubbo/、META-INF/services/

2. 解析文件内的别名和类名称，并缓存别名和类名的映射

   PS: 含有@Adaptive的类单独缓存到cacheAdaptiveClass，不会到扩展集合，用于后续的getAdaptiveExtension()

3. 当获取指定别名的扩展时，会从缓存中获取类，并反射无参构造实例，并缓存实例


