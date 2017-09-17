---
published: false
layout: post
title: "Reducing ongoing cost in AWS"
date: 2017-05-05 06:20:58 +0000
comments: false
categories: AWS, DevOps, Cost, Reduce Cost, EC2, Instances 
---

One of the main drivers for using a public cloud to host your applications is cost. You don't have to install, maintain or manage your own datacentre and you can run as many machines as you want. However, the cost does add up. 

In this post I will discuss some of the ways in which the cost of running applications in the AWS cloud can be reduced and controlled. Most of these topics can easily become the topic of their blog post so I'll just briefly touch on them in this post.  

* [Stop and start instances](#stop-start)
* [Reserved instances](#reserved-instances)
* [AWS EC2 Spot Market](#aws-spot)
* [Pick the correct size](#correct-size)
* [AWS Simple Storage Service](#s3)
* [Serverless computing](#serverless)
* [Keeping track of cost](#track-cost)
* [Avoid unexpected charges](#unexpected)

<a name="stop-start"></a>

### Stop and start intances

The first thing that comes to mind is that you probably have many virtual machines which aren't used 24/7. Especially ones that serve staff who work during normal office hours only. If you could stop those machines at night and start them again in the morning and leave them off over weekends, then you can massively save on the cost of running individual instances.

At the time of writing this a `t2.medium` AWS instance in Sydney cost `$0.064` to run for one hour. That doesn't sound like much but consider that there are `730 hours` in an average month. This means that the cost of running one `t2.medium` instance for a month is `$46.72`. If you only run the machine from 8am to 6pm every week day there are only `216 hours` in an average month, which means the `t2.medium` costs only `$13.82`. That is a saving of around `70%`!

For that matter stop and start anything you won't need for half a day or more. 

There are a number of free *stop and start your AWS instances* solutions on the web, but none of them matched our requirements so I coded a lambda function to manage our EC2 instances. It supports multiple regions and timezones, as well as complex schedules based on a list of cron expressions. It also supports detaching and reattaching instances that are part of a scale group, with RDS support coming soon. See it on github:  (TODO: Reference it on github)

<a name="reserved-instances"></a>

### Reserved Instances  <small><sup><small>[doc link][reserved_instances]</small></sup></small>

If you know that you will need a minmum amount of compute power 24 hours a day then reserving instances is an excellent way to bring down your overall AWS expense. It works by reserving some level of compute for a set period of time. Any instances you start that match the properties of the reserved instance will run at the reserved instance cost. For example, if you reserve two `t2.micro` instances for a year, then any `t2.micro` you start up during that year will run at the reserved price, as long as there are no more than two instances running at any time. A third instance will again run at the on-demand price.

> <small>Reserved Instances are not refundable, however they can be sold.</small>  

There are three main types of reserved instances.

**Standard Reserved Instances**
Standard Reserved Instances offers the most savings, up to 60% on a three year reservation, but is not as flexible as the other two. It does not, for example, allow you to change the operating system or instance type but you can change the instance size.


**Convertable Reserved Instances**
Convertable Reserved instances are more flexible than Standard Reserved instances but does not offer the same amount of savings. They allow changing of the reserved instance properties as long as the change results in the creation of an instance of equal or greater value. So no downgrading.


**Scheduled Reserved Instances**
If you can predict a time when your services will be in much higher demand then Scheduled Reserved Instances gives you the ability to reserve instances at a lower price for that time in the future. This allows you reserve the instances for a specified time window and instances will run at the reduced rate when started in this time window. 


<a name="aws-spot"></a>

### AWS EC2 Spot Market <small><sup><small>[doc link][spot_market]</small></sup></small>

The `spot market` offers considerable savings for instances by allowing you to bid how much you are willing to pay for an instance to be up. Spot instances are massively cheaper than on-demand instances and can bring savings of 50%-90%. 

In the spot market you bid some amount above the current `spot price` for an  EC2 instance, to indicate how much you are willing to pay for an instance of that size. You can then run an instance at the current spot price as long as your `bid` remains above the current spot price. If the spot price rises about your bid then AWS will send the instance a two minute termination warning before terminating it.

The current spot price is significantly lower than on demand prices and if you can live with instances suddenly terminating or can persist the instance state in less than two minutes then using the spot market is a great way to bring down EC2 cost. 


<a name="correct-size"></a>

### Pick the correct size 

Picking the right EC2 instance size is very important since that directly affects the per-hour price of running that virtual machine. This means if you run your application on an overly large EC2 instance then you will be charged for capacity you do not need. It is therefore important to architect applications so that processes that require a lot of processing power or memory is separated from those that can get away with less. A microservice architecture will give you the flexibility to pick instances that closely match your processing power needs. 

The cost of an EC2 instance is, in most cases, double when scaling double vertically so you are better off if you can get away with two small instances part of the time, instead of one large instance all of the time. 

<a name="s3"></a>

### Use S3 (Simple Storage Service) <small><sup><small>[doc link][s3]</small></sup></small>

S3 is a highly durable object store which you can use to store just about anything. You can serve whole static websites from S3 which removes the need to run a webserver on an EC2 instance. 

Cost of data transfer from an EC2 instances to the internet is just about the same as directly from an S3 bucket, but storing data on an EC2 instance is [significantly more expensive.][elastic_storage] It makes sense to, even if you run a webserver on an EC2 instance, to store the website static content and access logs in S3 rather than on the EC2 instance.

For data that will be accessed rarely, as in long term backups, move the data from S3 to [AWS Glacier Storage][glacier]. It is considerably cheaper to store data in Glacier - if you know it will be rarely retrieved. 

<a name="serverless"></a>

### Serverless computing <small><sup><small>[doc link][lambda]</small></sup></small>

There is a lot of excitement in this area at the moment. It is called serverless computing since you don't have to think about servers when implementing services. You just consider the individual small service and what it does, AWS handles the rest. 

Applications often have some services that run for very short periods of time and are called relative infrequently. These are often best implemented as AWS lambda functions rather than hosting them on an EC2 instance. Take the following simple example: 

Say you have a number of stateless [REST][rest] services which you want to expose on the internet. These services typically respond in one to two seconds and do some calculations or processing of data before returning the response to the client.  

##### Traditional approach
The traditional approach would be to host these using some backend server application running on a machine. This would usually involve a reverse proxy to do SSL termination, limiting access to specific services and protecting the server from direct internet access. Further you would duplicate the setup to ensure the services are available even if one of the machines should fail. Then due to load testing and known capacity requirements you may have multiple backend servers and spread the load accross them. This quickly grow to a considerable number of EC2 instances and mounting cost.

> <small> You always have a couple of EC2 instances running </small>

##### AWS Lambda
With lambda's you would implement the services in either Java, C#, Python or Javascript then upload them it to AWS. You would configure [AWS API Gateway][api_gateway] to expose the REST API to the outside world and forward any requests to the lamda functions.

The lambda's scale automatically and are automatically resilient so you don't have to worry about load balancing, scaling and duplicating deployments across regions. By implementing these services as individual lamda functions you no longer need EC2 instance running all the time.

With lamdba's you only pay for the time the code is actually running. If there are no requests to the service then it costs nothing to host the application. This could result in tremendous savings during quiet periods. 

> <small> With lamdas you can scale all the way to zero </small>

<a name="track-cost"></a>

### Keeping track of cost <small><sup><small>[doc link][aws_billing]</small></sup></small>
AWS provide a billing and cost management service which allows you to view and keep track of the total cost and also break the cost down to individual resources. You can see which parts of the infrastructure contribute most to the overall cost, which enables you to plan how you want to manage expenses.

You can set up alarms to be notified if specified thresholds are exceeded and set up budgets to visualise the status of spending on different aspects of the infrastructure. The cost visualisation feature allows you to see the status of the budget and estimate how much your spending will be. Budget amounts can also act like thresholds so that alerts can be sent should budgeted amount be exceeded.

You can set up cost and usage reports which will be published as CSV file and updated daily into a specified S3 bucket. You can choose to break down cost and usage by:

* Hour or Month
* Product or resource
* Custom Tags

<a name="unexpected"></a>

### Avoid unexpected charges <small><small>[doc link][unexpected_charges]</small></small>

When trying to minimize the cost of running your applications in AWS it is important to be aware of things which may attract some unexpected charges. To avoid this carefully read the pricing structure of services you are planning to use, some services are free and others are not. Some are surprisingly costly, take the NAT Gateway as a example.

The NAT Gateway enables your services to make outbound internet connections and receive the replies. It is highly available with redundancies in the same availability zone. It is can also handle burst bandwidth of up to 10Gbps. Share a NAT Gatway between multiple private networks unless you really need additional bandwith. If you don't need highly available outbound internet access then use a NAT Instance instead. Read more about NAT Gateways vs NAT Instances [here](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-comparison.html) 

Another service you may be surprised by is the Application Load Balancer. Here you may be tempted to use a load balancer per layer in your architecture. It is important to remember that the pricing structure of the Application Load Balancer is based on both time and usage.





### In Closing

There are numerous ways to keep track of and limit your spending in the AWS cloud. Some are easy and other requires some work to set up and use. With careful consideration and planning you can run your appilications and systems in cloud in very efficiently. You can scale resources as you need them and scale back down again.

Carefully plan the size of your eC2 instances and stop them when they are not required then start them again when they are. Reserve capacity for instances the base instances which will be up and running most of the time and use the spot market when scaling out to get the lowest possible price for the instance. Futher, use lambda's to reduce the cost of hosting microservices and use S3 for storing and serving static content.

One thing to note is that the cost of AWS resources is either per capacity or per usage. EC2 is an example where you pay for the capacity whether you use it or not, and S3 and lambda are an examples of where you pay only for what you used during the month. You can optimse your AWS spending by balancing when you need the sustained capacity and when a per usage model works best.


### Links

* [AWS Cost optimization](https://aws.amazon.com/pricing/cost-optimization/)
* [Avoid unexpected charges](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/checklistforunwantedcharges.html)


[reserved_instances]: https://aws.amazon.com/ec2/pricing/reserved-instances/
[spot_market]: https://aws.amazon.com/ec2/spot/
[s3]: https://aws.amazon.com/s3/
[elastic_storage]:https://aws.amazon.com/ebs/pricing/
[lambda]:https://aws.amazon.com/lambda/details/
[rest]:https://en.wikipedia.org/wiki/Representational_state_transfer
[api_gateway]:https://aws.amazon.com/api-gateway/
[aws_billing]:http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-what-is.html
[unexpected_charges]:http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/checklistforunwantedcharges.html