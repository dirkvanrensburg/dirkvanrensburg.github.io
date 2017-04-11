---
published: true
layout: post
title: "Hazelcast, JCache and Spring Boot"
date: 2017-02-17 08:09:39 +1100
comments: true
categories: hazelcast, spring-boot, jcache, java
---

[Hazelcast][hazelcast] is an in memory data grid which enables data sharing between nodes in a server cluster along with a full set of other data grid features. It also 
implements the [JCache (JSR107)][jcache] caching standard so it is ideal for building a data aggregation service.

In this post I'll go through the motions of adding Hazelcast to a Spring Boot REST application and resolving the issues until we have a functioning REST service 
with its response cached in Hazelcast via JCache annotations.

> <small>**TLDR;** I suggest reading the post to understand the eventual solution, but if you are impatient see the solution on github:
<br/>
* [hazelcast-jcache option 1 and 2](https://github.com/dirkvanrensburg/basic-spring-boot-rest/tree/hazelcast-jcache)
and<br/>
* [hazelcast-jcache option 3](https://github.com/dirkvanrensburg/basic-spring-boot-rest/tree/hazelcast-jcache-option3)</small>

> <small>**UPDATE 1:** It seems that this issue will be resolved soon due to the hard work of [@snicoll](https://github.com/snicoll) over at Spring Boot and the Hazelcast community. See the issues:
>
> * https://github.com/spring-projects/spring-boot/issues/8467
> * https://github.com/hazelcast/hazelcast/issues/10007
> * https://github.com/hazelcast/hazelcast/pull/9973
</small>

> <small>**UPDATE 2:** This problem described in this post was fixed in Spring Boot release 1.5.3. Check [this repository](https://github.com/dirkvanrensburg/hazelcast-springboot-jcache) for a clean example based on Spring boot 1.5.3. 
> I am leaving the post here since it is still interesting due to the different ways the problem could be worked around.</small>

### Versions 

Dependency  | Version
------------|-----------
Spring Boot | 1.5.1
Hazelcast   | 3.7.5

<br/>

### Spring boot REST

I am going to assume a working knowledge of building REST services using Spring Boot so I won't be going into too much detail here. Building a REST service in Spring is really easy and
 a quick Google will bring up a couple of tutorials on the subject.

This post will build on top of a basic REST app found on [github](https://github.com/dirkvanrensburg/basic-spring-boot-rest/tree/master). If you clone that you should be able to
follow along.

### Adding Hazelcast

To add Hazelcast to an existing Spring Boot project is very easy. All you have to do is add a dependency on Hazelcast, provide Hazelcast configuration and start using it.

#### Step 1
For maven add the following dependencies to your project `pom.xml` file:

``` XML Hazelcast dependencies in pom file
         <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast</artifactId>
        </dependency>

        <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast-spring</artifactId>
            <version>${hazelcast.version}</version>
        </dependency>
```

#### Step 2
I left the following Hazelcast configuration empty to keep things simple for now. Hazelcast will apply defaults for all the settings if you leave the configuration empty.
You can either provide a `hazelcast.xml` file on the classpath (e.g. `src/main/resources`)

``` XML XML expample of default Hazelcast configuration 
    <hazelcast xsi:schemaLocation="http://www.hazelcast.com/schema/config hazelcast-config-3.7.xsd"
           xmlns="http://www.hazelcast.com/schema/config"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    </hazelcast>
```

or provide a `com.hazelcast.config.Config` bean by means of Spring Java configuration like this:

``` Java Java example of default Hazelcast configuration
    @Configuration
    public class HazelcastConfig {

        @Bean
        public Config getConfig() {
            return new Config();
        }
    }
```

> <small>The hazelcast config can also be externalised from the application by 
passing the `-Dhazelcast.config` system property when starting the service.</small>

#### Step 3

Hazelcast will not start up if you start the application now. The Spring magic happens because of the 
`org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration` configuration class which is conditionally loaded by Spring 
whenever the application context sees that:

  1. `HazelcastInstance` is on the classpath
  + There is an unresolved dependency on a bean of type `HazelcastInstance`

To start using Hazelcast let's create a special service that will wire in a Hazelcast instance. The service doesn't do anything since it exists only to illustrate how
Hazelcast is configured and started by Spring.

``` Java Illustrate starting Hazelcast by adding a dependency
    @Service
    public class MapService {

        @Autowired
        private HazelcastInstance instance;

    }
```

If you start the application now and monitor the logs you will see that Hazelcast is indeed starting up. You should see something like:

    [LOCAL] [dev] [3.7.5] Prefer IPv4 stack is true.                                                                                             
    [LOCAL] [dev] [3.7.5] Picked [192.168.1.1]:5701, using socket ServerSocket[addr=/0:0:0:0:0:0:0:0,localport=5701], bind any local is true   
    [192.168.1.1]:5701 [dev] [3.7.5] Hazelcast 3.7.5 (20170124 - 111f332) starting at [192.168.1.1]:5701                                     
    [192.168.1.1]:5701 [dev] [3.7.5] Copyright (c) 2008-2016, Hazelcast, Inc. All Rights Reserved.                                             
    [192.168.1.1]:5701 [dev] [3.7.5] Configured Hazelcast Serialization version : 1                                                            

and a bit further down:

    Members [1] {                                                               
        Member [192.168.1.1]:5701 - f7225da2-a428-4849-944f-43abfb12063a this 
    }                                                                           

> <small> This is great! Hazelcast running with almost no effort at all!</small>

### Add JCache

Next we want to start using Hazelcast as a JCache provider. To do this add a dependency on `spring-boot-starter-cache` in your pom file.

``` XML Spring Boot Caching dependency in pom file 
    <dependency>
        <groupId>org.springframework.boot</groupId>o
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
```

Then, in order to use the annotations, add a dependency on the JCache API 

``` XML JCache dependency in pom file
    <dependency>
        <groupId>javax.cache</groupId>
        <artifactId>cache-api</artifactId>
    </dependency>
```

Finally to tell Spring to now configure caching add the `@EnableCaching` annotation to the Spring boot application class. The Spring boot application class is the one
that is currently annotated with `@SpringBootApplication` 

``` Java Enable Caching 
    @SpringBootApplication
    @EnableCaching
    public class Application {
```
Now something unexpected happens. Starting the application creates two hazelcast nodes which, if your local firewall allows multicast, join up to form a cluster. If multicasting works then you should see 
the following:


    Members [2] {
        Member [192.168.1.1]:5701 - 18383f04-43ac-41fc-a2bc-cd093a9706b6 this
        Member [192.168.1.1]:5702 - b654cb85-7b59-489d-b599-64ddd2dc0730
    }

    2017-02-16 08:41:46.154  INFO 14141 --- [ration.thread-0] c.h.internal.cluster.ClusterService      : [192.168.1.1]:5702 [dev] [3.7.5] 

    Members [2] {
        Member [192.168.1.1]:5701 - 18383f04-43ac-41fc-a2bc-cd093a9706b6
        Member [192.168.1.1]:5702 - b654cb85-7b59-489d-b599-64ddd2dc0730 this
    }

This is saying that there are two Hazelcast nodes running; One on port `5701` and another on port `5702` and they have joined to form a cluster.

> <small> This is an unexpected complication, but lets ignore the second instance for now.</small>

### Caching some results

Let's see if the caching works. Firstly we have to provide some cache configuration. Add the following to the `hazelcast.xml` file.

``` XML Hazelcast Cache configuration 
    <cache name="sumCache">
        <expiry-policy-factory>
            <timed-expiry-policy-factory expiry-policy-type="CREATED"
                                         duration-amount="1"
                                         time-unit="MINUTES"/>
        </expiry-policy-factory>
    </cache>
```

Next, to start caching change the `sum` function in the `CalculatorService` to:


``` Java Cached Sum Service
    @CacheResult(cacheName="sumCache")
    @RequestMapping(value = "/calc/{a}/plus/{b}", method = RequestMethod.GET)
    public CalcResult sum(@PathVariable("a") Double a, @PathVariable("b") Double b) {
         System.out.println(String.format("******> Calculating %s + %s",a,b));
     
         return new CalcResult(a + b);
    }
```

1. On `line 1` we added the `@CacheResult` annotation to indicate that we want to cache the result of this function and want to place them in the cache called `sumCache` 
+ On `line 4` we print a message to standard out so we can see the caching in action

Start the application and call the `CalculatorService` to add two numbers together. You can use `curl` or simply navigate to the following URL in a browser window

    http://localhost:8080/calc/200/plus/100 

> <small> **Note:** The port could be something other than 8080. Check the standard out when starting the application for the correct port number</small>

You'll get the following response

    {"result":300.0}

and should see this output in standard out:

    ******> Calculating 200.0 + 100.0

Subsequent calls to the service will not print this line,  but if you try again after a minute you'll see the message again.


> So caching works! But what about the second HazelcastInstance ?


### Only one instance

So you may think that the second Hazelcast instance is due to the example `MapService` we added earlier. To test that we can disable it by commenting the `@Service` annotation. 
You can even delete the class if you like, but the application will still start two Hazelcast nodes.

My guess as to what is going on is: When the application context starts, a Hazelcast instance is created by the JCache configuration due to the 
`@EnableCaching` annotation but this instance is *not* registered with the Spring context. Later on a new instance is created by the `HazelcastAutoConfiguration` 
which *is* managed by Spring and can be injected into other components. 


### Solution

I have found two solutions to the 'two instance problem' so far. Each with its own drawbacks

##### Option 1

I got the following idea from [Neil Stevenson](https://stackoverflow.com/users/5759723/neil-stevenson) over on [stackoverflow](https://stackoverflow.com/questions/42151600/hazelcast-and-jcache-in-spring-boot-creates-two-instances).

``` Java Disable the Hazelcast auto configuration
    @EnableAutoConfiguration(exclude = {
            // disable Hazelcast Auto Configuration, and use JCache configuration
        HazelcastAutoConfiguration.class, CacheAutoConfiguration
    })
```

**Drawback**: You can't use the Hazelcast instance created this way directly. Spring has no knowledge of it, so you can't get it wired in anywhere.


##### Option 2

This as the same effect as option 1 above except that you can use the instance. You have to name the hazelcast instance in the config:


    <instance-name>test</instance-name>


and then tell the Spring context to use it by getting it by name: 

``` Java Bring the instance into Spring
    @Bean
    public HazelcastInstance getInstance() {
        return Hazelcast.getHazelcastInstanceByName("test");
    }
```
**Drawback**: This relies on the order of bean creation so I can only say that it works in Spring Boot 1.5.1.


##### Option 3

The best solution so far is to set and instance name as in Option 2 above and then setting the `spring.hazelcast.config=hazelcast.xml` in the `application.properties` file.

see: [hazelcast-jcache-option3](https://github.com/dirkvanrensburg/basic-spring-boot-rest/tree/hazelcast-jcache-option3)

### Conclusion

I personally think that option 3 is the best approach. That gives you the best of both worlds with minimum configuration.

Spring Boot can be a bit magical at times and doesn't always do exactly what you would expect, but there is always a way to tell it to get out of the way and do 
it yourself. The people over at Spring are working hard to make everything 'just work' and I am confident that these things will be ironed out over time.


[hazelcast]: https://hazelcast.com/
[jcache]: https://jcp.org/aboutJava/communityprocess/final/jsr107/index.html
