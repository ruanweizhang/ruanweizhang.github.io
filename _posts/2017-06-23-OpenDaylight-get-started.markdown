---
layout: post
title: OpenDaylight startup
date: 2017-06-23 19:10:00 +08:00
tags: OpenDaylight, startup
---

# OpenDaylight Startup

This article is based on official tutorial [Startup Porject Archetype](https://wiki.opendaylight.org/view/OpenDaylight_Controller:MD-SAL:Startup_Project_Archetype). Since the official tutorial is out-of-date, I encounter some errors and this article is about how I solve them. Thanks for [Yochang](https://www.facebook.com/profile.php?id=100000181252042&fref=ts) and [Sesame](https://www.facebook.com/sesame.chen?fref=ts)'s contributions.

### develop environment
* OS: Ubuntu 16.04
* OpenDaylight: Boron-SR3

### part 1
In Ubuntu 16.04, after installing maven I couldn't find `~/.m2/settings.xml`, later I find out it is in ``/usr/share/maven/conf/``, copy it to `~/.m2`

```
cp /usr/share/maven/conf/settings.xml ~/.m2/settings.xml
```

In building `example` project, I choose `Boron-SR3`

```
mvn archetype:generate -DarchetypeGroupId=org.opendaylight.controller -DarchetypeArtifactId=opendaylight-startup-archetype -DarchetypeVersion=1.2.3-Boron-SR3 -DarchetypeRepository=https://nexus.opendaylight.org/content/repositories/public/ -DarchetypeCatalog=remote
```

### part 2

#### build `hello` project

In building `hello` project, I encounter this error

```
Failed to execute goal org.apache.maven.plugins:maven-archetype-plugin:3.0.1:generate (default-cli) on project standalone-pom: archetypeCatalog http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml' is not supported anymore. Please read the plugin documentation for details. -> [Help 1]
```

The reason is explained [here](http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html)

```
As of Maven Archetype Plugin 3.0.0 the archetype resolution has  
changed. It is not possible anymore to specify the repository   
via the commandline, but instead the repositories as already  
specified for Maven are used. This means that also the mirrors  
and proxies are respected, as well as the authentication on  
repositories.
```

solution is to lock the version of the plugin to `2.4`, run this command

```
mvn org.apache.maven.plugins:maven-archetype-plugin:2.4:generate -DarchetypeGroupId=org.opendaylight.controller -DarchetypeArtifactId=opendaylight-startup-archetype -DarchetypeRepository=opendaylight.release/ -DarchetypeCatalog=http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml -DarchetypeVersion=1.2.3-Boron-SR3
```
The tutorial is actually using `classPrefix` `Hello` instead of `hello`:

```
Define value for property 'classPrefix': Hello
```

build `hello` project using

```
mvn clean install
```

#### Implement `HelloWorld` RPC API
First create `HelloWorldImple.java` file then

```
cd hello/impl/src/main/resources/org/opendaylight/blueprint/

```
and revise `impl-blueprint.xml` file, then walk with the tutorial, if you have this error after `mvn clean build`:

```
[ERROR] src/main/java/org/opendaylight/hello/impl/HelloWorldImpl.java[1] (header) RegexpHeader: Line does not match expected header line of '^/[]+$'.
```

you can try with this:

```
mvn clean install -Dcheckstyle.skip
```

#### Test the 'hello-world' RPC via REST

![](/assets/images/2017/OpenDaylight-startup.png)
