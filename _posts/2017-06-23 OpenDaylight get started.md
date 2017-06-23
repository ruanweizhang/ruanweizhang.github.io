---
layout: post
title: OpenDaylight Get Started
date: 2017-06-23 15:11:24
---


In Ubuntu 16.04, after installing maven I couldn't find `~/.m2/settings.xml`, later I find out it is in ``/usr/share/maven/conf/``, copy it to `~/.m2`

```
cp /usr/share/maven/conf/settings.xml ~/.m2/settings.xml
```

In building `example` project, I choose `Boron-SR3`

```
mvn archetype:generate
-DarchetypeGroupId=org.opendaylight.controller\    
-DarchetypeArtifactId=opendaylight-startup-archetype\  
-DarchetypeVersion=1.2.3-Boron-SR3\  
-DarchetypeRepository=https://nexus.opendaylight.org/content/repositories/public/\  
-DarchetypeCatalog=remote
```

In building hello project, I encounter this error

```
Failed to execute goal org.apache.maven.plugins:maven-archetype-plugin:3.0.1:generate (default-cli) on project standalone-pom: archetypeCatalog 'http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml' is not supported anymore. Please read the plugin documentation for details. -> [Help 1]
```

The reason is explained [here](http://maven.apache.org/archetype/maven-archetype-plugin/archetype-repository.html):

```
As of Maven Archetype Plugin 3.0.0 the archetype resolution has  
changed. It is not possible anymore to specify the repository   
via the commandline, but instead the repositories as already  
specified for Maven are used. This means that also the mirrors  
and proxies are respected, as well as the authentication on  
repositories.
```

solution is to lock the version of the plugin to `2.4`, run this:

```
mvn org.apache.maven.plugins:maven-archetype-plugin:2.4:generate\  
-DarchetypeGroupId=org.opendaylight.controller\  
-DarchetypeArtifactId=opendaylight-startup-archetype \  
-DarchetypeRepository=opendaylight.release/ \  
-DarchetypeCatalog=http://nexus.opendaylight.org/content/repositories/opendaylight.snapshot/archetype-catalog.xml\  
-DarchetypeVersion=1.2.3-Boron-SR3
```

build `hello` project using:

```
mvn clean instal
```

#### License

This is licensed as [Creative Commons Attribution 4.0 International](http://creativecommons.org/licenses/by/4.0/). See the link for more information.
