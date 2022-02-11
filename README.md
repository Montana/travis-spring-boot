---
title: "Travis and Spring Boot"
created_at: Fri Feb 11 2022 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2022-02-11-springboot
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![Untitled-3](https://user-images.githubusercontent.com/20936398/153650476-6310d2c1-7692-45a7-962d-77f983f747c8.png)


In this series of tech blog Friday by Montana Mendy, we will learn how to run maven build goals, perform test coverage validation whether this be Coveralls, SonarCloud or Docker. Are you ready? I'm ready. Let's jump in.

<!-- more --> 

## Getting started

There's choices and choices. In this example I'll be using Maven, but you can obviously use Gradle as well. Next, you'll want to create a [SpringBoot project](http://start.spring.io/) either using http://start.spring.io or alternatively use your IDE. 

## Create your `.travis.yml` file

We have to enable your `.travis.yml` file and the way we do that is by the following: 

```yaml
language: java
jdk: oraclejdk8
```
Now this just invokes Travis into your project, this is sufficient enough for Travis to see that there's a build that needs to be triggered, how Travis is architected Travis will run `mvn test -B` for building the project. If Travis finds `mvnw wrapper` then it will be used like `./mvnw test -B.` as you can see it's a bit recursive. You can instruct Travis to run different commands in your `script:` hook in your `.travis.yml`.

## Adding Coverage Checks for Travis 

Add the Maven specific JaCoCo plugin to `pom.xml` with flags that define what is the desired code coverage percentage, classes, etc:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.7.9</version>
    <configuration>
        <excludes>
            <exclude>in/sivalabs/freelancerkit/entities/*</exclude>
            <exclude>in/sivalabs/freelancerkit/*Application</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <id>default-prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>default-prepare-agent-integration</id>
            <goals>
                <goal>prepare-agent-integration</goal>
            </goals>
        </execution>
        <execution>
            <id>default-report</id>
            <phase>verify</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>default-report-integration</id>
            <goals>
                <goal>report-integration</goal>
            </goals>
        </execution>
        <execution>
            <id>default-check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                 <rule implementation="org.jacoco.maven.RuleConfiguration">
                        <element>BUNDLE</element>
                        <limits>
                            <limit implementation="org.jacoco.report.check.Limit">
                                <counter>COMPLEXITY</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Classifiers 

To fix the classifier issue, the workaround is to add the classifier configuration to `spring-boot-maven-plugin` as follows:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <classifier>exec</classifier>
    </configuration>
</plugin>
```
Now let's take a look at our `travis.yml` file, and be on the lookout for the `script:` hook:

```yaml
language: java
jdk: oraclejdk8
  
script:
- ./mvnw clean install -B
```

You can see in our `script:` hook we've added `./mvnw vlean install -B` and that's going to be key. Now let's get a `.travis.yml` that has SonarCloud. Now remember to set your `environment variables` you grabbed from SonarCloud and also add them to Travis, here's a `.travis.yml` demonstrating this:

```yaml
language: java
jdk: oraclejdk8
  
env:
  global:
  - secure: (ENV_VARS_HERE)
  
addons:
  sonarcloud:
    organization: "montanamendy-from-travis"
    token:
      secure: $SONAR_TOKEN
  
script:
- ./mvnw clean install -B
- ./mvnw clean org.jacoco:jacoco-maven-plugin:prepare-agent package sonar:sonar
```

Now let's build a Dockerfile that mirrors essentially that `.travis.yml`: 

```Dockerfile
ROM frolvlad/alpine-oraclejdk8:slim
VOLUME /tmp
ADD target/freelancer-kit-0.0.1-SNAPSHOT.jar app.jar
RUN sh -c 'touch /app.jar'
ENV JAVA_OPTS="-Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8787,suspend=n"
EXPOSE 8080 8787
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -Dspring.profiles.active=docker -jar /app.jar" ]
```
That Dockerfile is going to grab `alpine-oraclejdk8:slim` the `slim` flag is making sure it's grabbinb the slim version. 

## Deploying 

Now let's say you're using Heroku for deployment, you'll need to grab all those credentials as well and add them to your `deploy:` hook.
