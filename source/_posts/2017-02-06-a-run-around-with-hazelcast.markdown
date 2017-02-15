---
published: true
layout: post
title: "Hazelcast, JCache and Spring Boot"
date: 2017-02-06 15:09:39 +1100
comments: true
categories:
---

[Hazelcast][hazelcast] is an in memory data grid which enables data sharing between nodes in a server cluster along with a full set of other data grid features. It also implements the [JCache (JSR107)][jcache] caching standard for Java so it makes the ideal option when you are building a data aggregation service.

I decided to add use Hazelcast for caching results from a set REST services, based on Spring MVC. It is fairly straightforward to add Hazelcast to a spring project, but I was surprised to learn that Spring Boot's autoconfiguration magic creates two Hazelcast instances when adding `@EnableCaching`.

In this post I'll go through the motions of adding Hazelcast to a Spring Boot REST application and resolving the issues until we have a functioning REST service with its response cached in Hazelcast via JCache annotations.


#### Versions and other things to note

Dependency  | Version
------------|-----------
Spring Boot | 1.5.1
Hazelcast   | 3.7.5


* Hazelcast configuration is done through `hazelcast.xml` file


### Spring boot REST

I am going to assume a working knowledge of building REST services using Spring Boot so I won't be going into too much detail here. Building a REST service in Spring is really easy and
 a quick Google will bring up a couple of tutorials on that subject.

This post will build on top of a basic REST app found on [github](https://github.com/dirkvanrensburg/basic-spring-boot-rest/tree/master)

### Adding Hazelcast

To add Hazelcast to an existing Spring Boot project is very easy. All you have to do is add a dependency on Hazelcast and provide Hazelcast configuration and start using it.


#### Step 1
For maven add the following dependencies to your project `pom.xml` file:

         <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast</artifactId>
        </dependency>

        <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast-spring</artifactId>
            <version>${hazelcast.version}</version>
        </dependency>


#### Step 2
I left the following Hazelcast configuration empty to keep things simple for now. Hazelcast will apply defaults for all the settings if you leave the configuration empty.
You can either provide a `com.hazelcast.config.Config` bean by means of Spring configuration like this:


    @Configuration
    public class HazelcastConfig {

        @Bean
        public Config getConfig() {
            return new Config();
        }

    }

or provide a 'hazelcast.xml' file on the classpath

    <hazelcast xsi:schemaLocation="http://www.hazelcast.com/schema/config hazelcast-config-3.7.xsd"
           xmlns="http://www.hazelcast.com/schema/config"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    </hazelcast>


> The hazelcast config can also be externalised from the application by 
> passing the `-Dhazelcast.config` system property when starting the service.

#### Step 3

Hazelcast will not start up if you start the application now. The Spring magic happens because of 
the `import org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration` configuration class which is conditionally loaded by Spring 
whenever the application context sees that:

  1. `HazelcastInstance` is on the classpath
  + There is an unresolved dependency on a bean of type `HazelcastInstance`

To start using Hazelcast lets create a special service that will wire in a Hazelcast instance. The service doesn't do anything since it exists only to show how
Hazelcast is configured and started by Spring.

    @Service
    public class MapService {

        @Autowired
        private HazelcastInstance instance;

    }

If you start the application now and monitor the logs you will see that Hazelcast is indeed starting up. You should see something like:

    [LOCAL] [dev] [3.7.5] Prefer IPv4 stack is true.                                                                                             
    [LOCAL] [dev] [3.7.5] Picked [192.168.148.1]:5701, using socket ServerSocket[addr=/0:0:0:0:0:0:0:0,localport=5701], bind any local is true   
    [192.168.148.1]:5701 [dev] [3.7.5] Hazelcast 3.7.5 (20170124 - 111f332) starting at [192.168.148.1]:5701                                     
    [192.168.148.1]:5701 [dev] [3.7.5] Copyright (c) 2008-2016, Hazelcast, Inc. All Rights Reserved.                                             
    [192.168.148.1]:5701 [dev] [3.7.5] Configured Hazelcast Serialization version : 1                                                            

and a bit further down:

    Members [1] {                                                               
        Member [192.168.148.1]:5701 - f7225da2-a428-4849-942f-43abeb12023a this 
    }                                                                           

### Add JCache


    <dependency>
        <groupId>javax.cache</groupId>
        <artifactId>cache-api</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>


    @EnableCaching
    public class Application {



### Two instances

### Only one instances

### Conclusion




1) Add hazelcast
According to spring docs here you add Hazelcast like this:

TODO: Find the docs

        <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast</artifactId>
            <version>3.7.5</version>
        </dependency>

test this and we see the hazelcast instance starting but it is the wrong version. This is because spring boot's pom is managing the dependency on hazelcast and you will either have to override this in your own dependency management section or override the version. The easiest is to just override the version in your own pom:

     <properties>
            <hazelcast.version>3.7.5</hazelcast.version>
     </properties>



2) Spring starter cache

   To magically enable caching and JCache use

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>



But now we have two hazelcast instances.....? are they the same? set a name in the hazelcast config to be sure...
Yes we can now see that it is trying to start the same instance.


Lets live with this for a while and see if we can get the cache to work

## Install and start the management center
 .. . . .

as we can see the cache is there and we can see the hits and misses - nice!


## Lets get rid of the duplicate hazelcast instance

To fix this we need to provide a cachemanager: ( write about how you figured this out ? )


First lets try as they describe here:

http://docs.hazelcast.org/docs/latest/manual/html-single/index.html#hazelcast-jcache

Hey that works we no don't have a second instance anymore - BUT it no longer caches!!


So lets try doing it this way:


    @Bean
    public com.hazelcast.spring.cache.HazelcastCacheManager cacheManager(HazelcastInstance instance) {
        return new com.hazelcast.spring.cache.HazelcastCacheManager(instance);
    }

but this means we have to import



        <dependency>
            <groupId>com.hazelcast</groupId>
            <artifactId>hazelcast-spring</artifactId>
            <version>${hazelcast.version}</version>
        </dependency>


This makes the cachemanager available and yay now we only have one hazelcast instance. And caching works!

Test it by making a call -> yes we are getting the value from the cache

check the mancenter -> What now we have a map instead of a cache? and our cache eviction configuration is ignored!


THIS DOES NOT WORK:



    @Bean
    public CacheManager cacheManager(HazelcastInstance instance) {

//        HazelcastCacheManager cacheManager = new com.hazelcast.spring.cache.HazelcastCacheManager(instance);
//
//        return cacheManager;

        CacheManager manager = Caching.getCachingProvider().getCacheManager();
        return manager;
    }



THIS DOES NOT WORK EITHER:

    @Bean
    public CacheManager cacheManager(CachingProvider provider) {

//        HazelcastCacheManager cacheManager = new com.hazelcast.spring.cache.HazelcastCacheManager(instance);
//
//        return cacheManager;



        CacheManager manager = provider.getCacheManager();
        return manager;
    }



[hazelcast]: https://hazelcast.com/
[jcache]: https://jcp.org/aboutJava/communityprocess/final/jsr107/index.html
