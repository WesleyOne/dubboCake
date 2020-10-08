---
sort: 3
title: Spring-XML配置
tags: Dubbo Spring XML 配置 源码入口
---

## 样例XML：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

        <dubbo:application name="dubbo-provider" />

        <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>

        <dubbo:protocol name="dubbo" port="20880" />

        <dubbo:provider loadbalance="random" />

        <bean id="helloServiceImpl" class="com.example.provider.service.HelloServiceImpl"/>

        <dubbo:service interface="org.example.api.HelloService" ref="helloServiceImpl" retries="0" timeout="10000">
                <dubbo:method name="say" retries="1" >
                        <dubbo:argument type="java.lang.String"/>
                </dubbo:method>
                <dubbo:method name="say" retries="2">
                        <dubbo:argument type="java.lang.String"/>
                        <dubbo:argument type="java.lang.String"/>
                </dubbo:method>
        </dubbo:service>

</beans>
```



1. XML配置查找NamespaceHandler（Spring知识点）

	- 查找dubbo相关的命名空间，比如样例中的`http://dubbo.apache.org/schema/dubbo`
	- 查找dubbo依赖jar包下定义的spring.handlers配置文件，里面配置了命名空间和命名空间处理器之间的关系。2.6.4版本如下：

	```
	http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
	http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
	```

2. 注册解析器
   
	- 使用Spring的XML配置，结合DubboNamespaceHandler注册各种标签的Bean定义解析器（BeanDefinitionParser）
   
3. 解析XML构建Bean定义
   - 在Spring启动过程的加载解析XML阶段，读取到dubbo相关标签时，会查找对应的解析器来构建指定的Bean定义（除了service和refrence是ServiceBean、工厂类ReferenceBean，其他对应的是XXConfig，ServiceBean和ReferenceBean是Dubbo和Spring整合的关键，更好的结合Sping本身的特性）
   - 实质都是DubboBeanDefinitionParser解析器，只是会根据不同的Bean定义进行判断设置特殊属性
   - 通用属性设置。获取Bean定义类的所有方法，然后判断是否set开头，是的话获取对应的get开头或者is开头的设置方法，设置方法也存在时，获取标签属性值，存在则设置到Bean定义的属性管理器里

4. Bean实例化
   
   - Spring默认非懒加载，在onRefresh中会将上述Bean定义实例化。
