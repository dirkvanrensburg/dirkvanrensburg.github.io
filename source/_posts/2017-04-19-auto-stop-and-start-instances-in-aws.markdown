---
published: false
layout: posts
title: "Auto stop and start instances in AWS"
date: 2017-04-19 06:20:58 +0000
comments: true
categories: 
---

One of the main drivers for using a public cloud to host your applications is cost. You don't have to install, maintain or manage your own datacentre and you spin up as many machines as 
you want. However, the cost of runningu a number of virtual machines add up and the cost of running the applications in the cloud can quickly add up and become a problem.

In this post I will discuss various ways in which the cost of running applications in the AWS cloud can be controlled and reduced.

*Stop virtual machines*
The first thing that comes to mind is that there are many virtual machines that you probably don't use 24/7. Especially ones that serve staff who work during normal office hours only. If you could stop those machines at night and start them again in the morning and leave them off over weekends, then you can massively save on the cost of running individual instances.

At the time of writing this a `t2.medium` AWS instance in Sydney cost `$0.064` to run for one hour. That doesn't sound like much but consider that there are `730 hours` in an average month. This means that the cost of running one `t2.medium` instance for a month is `$46.72`. If you only run the machine from 8am to 6pm every week day there are only `216 hours` in an average month, which means the `t2.medium` costs only `$13.82`. That is a saving of around `70%`!

TODO: Up until recently there was no way to easily schedule EC2 instances to be shutdown and then start again. Now AWS has provides a [lambda]() which you can use to do just this. You tag the EC2 instances you want to shut down and start up again and the lambda will do it for you.

TODO: At my company, [Avocado Consulting][] we use a python script I wrote called 'sentinel' to stop and start our instances in AWS. It supports mutliple regions and timezones, as well as complex schedules based on a list of cron expressions. It aslo supports detaching and reattaching instances that are part of a scale group. See it on github:  (TODO: Ensure sentinel evaluates the CRONs in the correct order.) Reference it on github


*Spot prices*
Apart from stopping and starting virtual machines, cost can be brought down further by using the TODO:( find name: AWS spot market) to get instances at a lower price. Explain how this works: 

* This has the drawback that the instance could be killed by AWS should the spot price rize above your reserve


*Cloud front*
AWS provide the marvelous S3 infrastructure to store static content such images and documents. However if these are frequently accessed like a image on a busy website then cost could quickly escalate. You can use cloud front to move frequently accessed files to internet edge locations so that webrequests never hit the s3 bucket itself.

* transfer from s3 to cloudfront is free (TODO: Check this)
* transfer from cloudfront is significantly cheaper (find numbers) 


*Serverless*
Use serverless services for tasks which do not require an always on process. For example as mentioned before serve static content from s3 and cloudfront. Use AWS lambdas to respond to webrequests which do not require significant processing time.

TODO: add more serverless


*Tracking cost*
Keep track of cost using AWS's built in reporting. 
* tag instances so cost can be broken down by type(webservers) or environment (testing)
* Create seperate accounts to restrict spending. For example only allow 't2.micro' instances in the AWS playground environment.
* Investigate how spending limits and alarms work.....


