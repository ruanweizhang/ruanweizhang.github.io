---
layout: post
title: Blueprint in Opendaylight
date: 2017-06-30 21:40:00 +08:00
tags: OpenDaylight, Blueprint
---

### 什么是Blueprint
Blueprint是Apache Aries底下的一个项目。Apache Aries project consists of a set of
pluggable Java components enabling an enterprise OSGi application programming
models, including:
* Asynchronous Services and Promises Specification
* **Blueprint Specification**
* JTA Transaction Services Specification
* JMX Management Model Specification (ODL也有用到JMX，之后再来研究这个)
* JNDI Services Specification
* JPA Service Specification
* Service Loader Mediator Specification
* Subsystem Service Specification

官网上的介绍：
Blueprint provides a dependency injection framework for OSGi and was standardized
by the OSGi Alliance in OSGi Compendium R4.2. It is designed to deal with the dynamic
nature of OSGi, where services can become available and unavailable at any time.
一句话总结：Blueprint就是为了解决OSGi里面的依赖关系。

#### Blueprint语法
* **bean** - an element that describes a Java object to be instantiated given a class name and optional constructor args and properties.
* **service** - advertises a bean as an OSGi service
* **reference** - imports a singleton OSGi service that implements a specified interface and/or satisfies a specified property filter
* **reference-list** - imports multiple OSGi services that implement a specified interface and/or satisfy a specified property filter

下面会有一些例子使用到这些语法
#### Blueprint XML
每份Blueprint的结构都必须是长这样的，xmlns是xml namespace的意思
```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0">
    ...
</blueprint>
```

#### 引入MA-SALservices: DataBroker, RpcProviderRegistry, NotificationPublishService

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0"
    odl:use-default-for-reference-types="true">

  <reference id="dataBroker" interface="org.opendaylight.controller.md.sal.binding.api.DataBroker" odl:type="default"/>
  <reference id="rpcRegistry" interface="org.opendaylight.controller.sal.binding.api.RpcProviderRegistry"/>
  <reference id="notificationService"
          interface="org.opendaylight.controller.md.sal.binding.api.NotificationPublishService"/>

</blueprint>
```
注意reference引入的都是interface

### Bean
Bean is an element that describes a Java object to be instantiated given a class
 name and optional constructor args and properties. 从这句话能看出来bean其实就是引入的一个class。但是为什么一个class要特意叫做bean呢，是不是多此一举？
 [hvhotcodes@stackoverflow](https://stackoverflow.com/questions/3295496/what-is-a-javabean-exactly)
 的回答是
> A JavaBean is a standard including:
> * All properties private (use getters/setters)
> * A public no-argument constructor
> * Implements Serializable.
Quote break

符合这个standard的class会有一些优点，有兴趣的可以再看下[杨博@知乎](https://www.zhihu.com/question/19773379)的回答。

下面是在Blueprint引入bean的一个例子，从org.opendaylight.spark.impl包下引入HelloProvider class，并取名为provider.
 ```xml
 <?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0">
   <bean id="provider" class="org.opendaylight.spark.impl.HelloProvider" />
</blueprint>
 ```

 在我上一篇文章`Hello Project`里面，我的impl-blueprint.xml是长下面这样的，可以看到引入了两个interface,
 DataBroker, RpcProviderRegistry,然后定义bean的时候将这两个作为argument参数传进去了

 ```xml
 <blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
  xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0"
  odl:use-default-for-reference-types="true">

  <reference id="dataBroker"
    interface="org.opendaylight.controller.md.sal.binding.api.DataBroker"
    odl:type="default" />

  <reference id="rpcRegistry"
    interface="org.opendaylight.controller.sal.binding.api.RpcProviderRegistry"/>

  <bean id="provider"
    class="org.opendaylight.spark.impl.HelloProvider"
    init-method="init" destroy-method="close">
    <argument ref="dataBroker" />
    <argument ref="rpcRegistry" />
 </bean>

</blueprint>
 ```

 参数传进去之后，对应的HelloProvider的constructor代码是：

 ```java
 public HelloProvider(final DataBroker dataBroker, RpcProviderRegistry rpcProviderRegistry) {
    this.dataBroker = dataBroker;
    this.rpcProviderRegistry = rpcProviderRegistry;
}
 ```
 这样HelloProvider就可以construct一个instance了。

### Advertising services
 
如果想要将自己实现的interface (*也就是bean*) 作为一个service注册到MD-SAL，供其他人reference的话，这个过程就叫advertise service

 ```xml
 <?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0">

  <bean id="helloSerivceImpl" class="org.opendaylight.app.HelloSerivceImpl">
    <!-- constructor args -->
  </bean>

  <service ref="helloSerivceImpl" interface="org.opendaylight.app.HelloSerivce" odl:type="default"/>

</blueprint>
 ```
 上面实现的是将HelloSerivceImpl所实现的HelloSerivce interface advertise出去。在ODL命名规则里面，XXXImpl通常是XXX的实现，这样XXXImpl通常就是实现的方法，而XXX通常就是interface的定义. 前面提到，service是将一个bean advertise出去，这样其他人就可以像上面一样利用reference引用了

 ```xml
 <reference id="helloSerivceImpl"
   interface="org.opendaylight.app.HelloSerivce"
   odl:type="default" />
 ```

 ### MD-SAL Bluerint extensions
Opendaylight provides 2 blueprint extensions to make it easier to register and consume global MD-SAL RPC services.。 RPC service有三种：
 * Global service: There is only one instance of a Global Service per controller instance (Note that a controller instance can consist of a cluster of controller nodes).
 * Routed service: There can be multiple instances (implementations) of a service per controller instance
 * Mounted service: Mounted service is a special type of services, which could be nested at well-known points in the overall data tree.

blueprint专门为RPC提供的两种extension分别是global service和routed service
#### Global RPCs
use `rpc-implements` element in provider to register a service implementation

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0">

  <bean id="fooRpcService" class="org.opendaylight.app.FooRpcServiceImpl">
    <!-- constructor args -->
  </bean>

  <odl:rpc-implementation ref="fooRpcService"/>

</blueprint>
```
use `odl:rpc-service` in consumer to use the service implementation

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0">

  <odl:rpc-service id="fooRpcService" interface="org.opendaylight.app.FooRpcService"/>

  <bean id="bar" class="org.opendaylight.app.Bar">
    <argument ref="fooRpcService"/>
  </bean>

</blueprint>
```
这种做法跟上面的advertising services有什么实质上的区别，目前还不看出来。先留个坑，以后弄清楚了再来补。

#### Routed RPCs
use `odl:routed-rpc-implementation` element

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0">

  <bean id="fooRoutedRpcService" class="org.opendaylight.app.FooRoutedRpcServiceImpl">
    <!-- constructor args -->
  </bean>

  <odl:routed-rpc-implementation id="fooRoutedRpcServiceReg" ref="fooRoutedRpcService"/>

  <bean id="bar" class="org.opendaylight.app.Bar">
    <argument ref="fooRoutedRpcServiceReg"/>
  </bean>

</blueprint>
```

#### NotificationListener
ODL还有对notification的extension。notification是ODL messaging pattern之一，不属于rpc。下面的做法是先实作出FooNotificationListener，并且作为bean引入到blueprint，再使用`odl:notification-listener` element注册到MD-SAL NotificationService，从而接收yang notifications.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
                 xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0">

  <bean id="fooListener" class="org.opendaylight.app.FooNotificationListener">
    <!-- constructor args -->
  </bean>

  <odl:notification-listener ref="fooListener"/>

</blueprint>
```

#### Application Configuration

ODL还提供一种extension叫`odl:clustered-app-config`的element，主要是为了能够configure一些applicaiton-defined configuration yang data. yang文件里面定义的data分两种，`config false`的是operational data，`config true`的是configurational data。这个extension就是为了能够从data store里面获得并改变configuration data。有兴趣的可以去看下[Toaster Tutorial Step by Step](https://wiki.opendaylight.org/view/OpenDaylight_Controller:MD-SAL:Toaster_Step-By-Step)的`part 7：Part 7: Adding configuration for the Toaster provider application`, 不过因为过时了里面很多错请做好心理准备==!
