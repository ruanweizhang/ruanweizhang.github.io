---
layout: post
title: OpenDaylight startup
date: 2017-06-23 19:10:00 +08:00
tags: OpenDaylight, Hello, Project
---


This article is based on official tutorial [Startup Project Archetype](https://wiki.opendaylight.org/view/OpenDaylight_Controller:MD-SAL:Startup_Project_Archetype). Since the official tutorial is out-of-date, I encounter some errors and this article is about how I solve them. Thanks for [Yochang](https://www.facebook.com/profile.php?id=100000181252042&fref=ts) and [Sesame](https://www.facebook.com/sesame.chen?fref=ts)'s contributions.

### environment
* OS: Ubuntu 16.04
* OpenDaylight: Boron-SR3

### part 1
In Ubuntu 16.04, after installing maven I couldn't find `~/.m2/settings.xml`, later I find out it is in ``/usr/share/maven/conf/``, copy it to `~/.m2`

```bash
cp /usr/share/maven/conf/settings.xml ~/.m2/settings.xml
```

In building `example` project, I choose `Boron-SR3`

```bash
mvn archetype:generate -DarchetypeGroupId=org.opendaylight.controller -DarchetypeArtifactId=opendaylight-startup-archetype -DarchetypeVersion=1.2.3-Boron-SR3 -DarchetypeRepository=https://nexus.opendaylight.org/content/repositories/public/ -DarchetypeCatalog=remote
```

### part 2

#### build `hello` project

In building `hello` project, I encounter this error

```bash
Failed to execute goal org.apache.maven.plugins:maven-archetype-plugin:3.0.1:generate (default-cli) on project standalone-pom: archetypeCatalog http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml' is not supported anymore. Please read the plugin documentation for details. -> [Help 1]
```

The reason is explained [here](http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html)

```bash
As of Maven Archetype Plugin 3.0.0 the archetype resolution has changed. It is not possible anymore to specify the repository via the commandline, but instead the repositories as already specified for Maven are used. This means that also the mirrors and proxies are respected, as well as the authentication on repositories.
```

solution is to lock the version of the plugin to `2.4`, run this command

```bash
mvn org.apache.maven.plugins:maven-archetype-plugin:2.4:generate -DarchetypeGroupId=org.opendaylight.controller -DarchetypeArtifactId=opendaylight-startup-archetype -DarchetypeRepository=opendaylight.release/ -DarchetypeCatalog=http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml -DarchetypeVersion=1.2.3-Boron-SR3
```
The tutorial is actually using `classPrefix` `Hello` instead of `hello`:

```bash
Define value for property 'classPrefix': Hello
```

到这里我们只是有了一个符合maven结构的目录框架，想要运行还需要在project根目录下build `hello` project

```bash
mvn clean install
```
然后运行

```
./karaf/target/assembly/bin/karaf
```
看到下面的显示就说明成功了

```
opendaylight-user@root>
```
#### Implement `HelloWorld` RPC API
首先修改hello.yang, 定义一个叫hello-world的rpc，为了最后可以用REST的方式调用

```
vim ~/hello/api/src/main/yang/hello.yang
```

```
module hello {
    yang-version 1;
    namespace "urn:opendaylight:params:xml:ns:yang:hello";
    prefix "hello";

    revision "2015-01-05" {
        description "Initial revision of hello model";
    }
    rpc hello-world {
        input {
            leaf name {
                type string;
            }
        }
        output {
            leaf greeting {
                type string;
            }
        }
    }
}
```
然后建立 `HelloWorldImple.java`

```
vim ~/hello/impl/src/main/java/org/opendaylight/hello/impl/HelloWorldImpl.java
```

```
package org.opendaylight.hello.impl;

import java.util.concurrent.Future;

import org.opendaylight.yang.gen.v1.urn.opendaylight.params.xml.ns.yang.hello.rev150105.HelloService;
import org.opendaylight.yang.gen.v1.urn.opendaylight.params.xml.ns.yang.hello.rev150105.HelloWorldInput;
import org.opendaylight.yang.gen.v1.urn.opendaylight.params.xml.ns.yang.hello.rev150105.HelloWorldOutput;
import org.opendaylight.yang.gen.v1.urn.opendaylight.params.xml.ns.yang.hello.rev150105.HelloWorldOutputBuilder;
import org.opendaylight.yangtools.yang.common.RpcResult;
import org.opendaylight.yangtools.yang.common.RpcResultBuilder;

public class HelloWorldImpl implements HelloService {

    @Override
    public Future<RpcResult<HelloWorldOutput>> helloWorld(HelloWorldInput input) {
        HelloWorldOutputBuilder helloBuilder = new HelloWorldOutputBuilder();
        helloBuilder.setGreeting("Hello " + input.getName());
        return RpcResultBuilder.success(helloBuilder.build()).buildFuture();
    }

}
```
 然后注册rpc。只有在MD-SAL注册了，从外面调用一个rpc的时候，MD-SAL才知道去找哪个provider.注册分两步，分别要修改impl-blueprint.xml和HelloProvider.java

```
vim hello/impl/src/main/resources/org/opendaylight/blueprint/impl-blueprint.xml

```

```
 <?xml version="1.0" encoding="UTF-8"?>
 <!-- vi: set et smarttab sw=4 tabstop=4: -->
 <!--
 Copyright © 2016 Cisco Systems and others. All rights reserved.
 
 This program and the accompanying materials are made available under the
 terms of the Eclipse Public License v1.0 which accompanies this distribution,
 and is available at http://www.eclipse.org/legal/epl-v10.html
 -->
 <blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
   xmlns:odl="http://opendaylight.org/xmlns/blueprint/v1.0.0"
   odl:use-default-for-reference-types="true">
  
   <reference id="dataBroker"
     interface="org.opendaylight.controller.md.sal.binding.api.DataBroker"
     odl:type="default" />
 
   <reference id="rpcRegistry"
     interface="org.opendaylight.controller.sal.binding.api.RpcProviderRegistry"/>'''
 
   <bean id="provider"
     class="org.opendaylight.spark.impl.HelloProvider"
     init-method="init" destroy-method="close">
     <argument ref="dataBroker" />
     <argument ref="rpcRegistry" />
  </bean>
 
 </blueprint>
```
保存退出，接着

```
vim hello/imp/src/main/java/org/opendaylight/hello/impl/HelloProvider.java
```

```
/*
 * Copyright © 2016 Cisco Systems and others.  All rights reserved.
 *
 * This program and the accompanying materials are made available under the
 * terms of the Eclipse Public License v1.0 which accompanies this distribution,
 * and is available at http://www.eclipse.org/legal/epl-v10.html
 */
package org.opendaylight.hello.impl;

import org.opendaylight.controller.md.sal.binding.api.DataBroker;
import org.opendaylight.controller.sal.binding.api.RpcProviderRegistry;
import org.opendaylight.controller.sal.binding.api.BindingAwareBroker.RpcRegistration;
import org.opendaylight.yang.gen.v1.urn.opendaylight.params.xml.ns.yang.hello.rev150105.HelloService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class HelloProvider {

    private static final Logger LOG = LoggerFactory.getLogger(HelloProvider.class);

    private final DataBroker dataBroker;
    private final RpcProviderRegistry rpcProviderRegistry;
    private RpcRegistration<HelloService> serviceRegistration;

    public HelloProvider(final DataBroker dataBroker, RpcProviderRegistry rpcProviderRegistry) {
        this.dataBroker = dataBroker;
        this.rpcProviderRegistry = rpcProviderRegistry;
    }

    /**
     * Method called when the blueprint container is created.
     */
    public void init() {
    	serviceRegistration = rpcProviderRegistry.addRpcImplementation(HelloService.class, new HelloWorldImpl());
        LOG.info("HelloProvider Session Initiated");
    }

    /**
     * Method called when the blueprint container is destroyed.
     */
    public void close() {
    	serviceRegistration.close();
        LOG.info("HelloProvider Closed");
    }
}
```
然后在hello project下再install一次

```
mvn clean install
```
如果遇到了下面这个错误的话

```bash
[ERROR] src/main/java/org/opendaylight/hello/impl/HelloWorldImpl.java[1] (header) RegexpHeader: Line does not match expected header line of '^/[]+$'.
```

可以尝试这个命令

```bash
mvn clean install -Dcheckstyle.skip
```

#### Test the `hello-world` RPC via REST

![](/assets/images/2017/OpenDaylight-startup.png)
