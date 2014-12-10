---
layout: post
title: "Supplementing metrics in AppDynamics"
date: 2014-12-10 11:39:17 +1100
comments: false
published: false
categories:

#NOTE THIS IS A REWORK OF ANOTHER POST - PUBLISHED THE OTHER ONE INSTEAD
---

AppDynamics gathers information about system performance using many different sensors. It has agents for Java, .Net, PHP to name the big ones, but there is also the [Standalone Machine agent][appd_machine_agent]. The machine agent is a more generic approach to collect metrics and it can be extended to gather any kind metrics you have access to. You can write a custom [Monitoring extension][appd_monitor_extension] to collect metrics from any source for example running custom queries on your database to count orders in progress, calling a legacy system's services using its proprietary socket based protocol. Anything that can be monitored programmatically can potentially be turned into metrics for AppDynamics by the Standalone Machine agent.

We used this ability of sending metrics to the AppDynamics controller to help one of our customers to build a very specific dashboard, which would not be possible otherwise. It involved calling the AppDynamics Controller REST API to extract metrics captured by the Java agent summarising it and then sending it back as new metrics. These new metrics could then be used to display information from different time ranges on the same dashboard using one status light widget.

*This post is a shortened version of [this more detailed post](http://dirkvanrensburg.github.io/blog/2014/11/28/advanced-metrics-collecting-in-appdynamics/) by one of our consultants.*

### The problem

The client had an existing implementation of CA Wily Introscope a.k.a. [CA APM][ca_apm] and they wanted a particular Wily dashboard implemented using AppDynamics. This particular dashboard used status lights to convey information about how recently a client of theirs accessed their system. These client system should ideally connect all the time and the dashboard was used to proactively determine when a client is having problems connecting.

The Introscope dashboard looked like the following mockup. 

{% img center /images/ext_metrics_complex_dashboards_appd/wily_dashboard.png "Introscope Dashboard" %}

Each light represents a client and the colours indicate how recently they accessed the system.

| __Status__ 	|        __Meaning__                         			|
| --------------| ------------------------------------------ 			|
| GREY		| No activity since 5am this morning           			|
| RED		| No activity in the last hour, but something since 5 am	|
| YELLOW	| No activity in the last 10 minutes, but some in the last hour	|
| GREEN		| Activity in the last 10 minutes			   	|

&nbsp;
Status lights in AppDynamics work much different to the Instroscope ones. In AppDynamics the status light is a visual representation of a health rule. The light is yellow if the health rule it is linked to is in a WARNING state, red for CRITICAL and green otherwise. This means the light will be green in the case where there is no data and that the AppDynamics status light has only three states compared to the four in Introscope.

Health rules in AppDynamics is evaluated over metrics captured during a specified window and these values are then compared to a baseline or some fixed threshold. This means that a single health rule cannot compare data from different points in time. A status light can be linked to only one health rule and we needed to create new metrics to condense the information into one.

### Collecting the information

The first step to solving the recreating the dashboard is to collect the information. In this case the information came from calls to a particular method on a [POJO][pojo]. In AppDynamics you can track calls to methods using [information points][appd_info_points], which collects metrics such as *Calls Per Minute* and *Average Response Time* for calls to a method. Once collected the information captured by an information point can be viewed using the AppDynamics controller.

{% img center /images/ext_metrics_complex_dashboards_appd/infopoint_metrics.png Information Point Metrics %}

### Extracting metrics from AppDynamics

Metrics can be extracted from the AppDynamics controller using the [REST API][appd_rest_api]. To extract a particular metric, in our case information point metrics, you will need the URL which uniquely identify the metric to the REST interface. You can retrieve this URL from the metric browser in the controller.

{% img center /images/ext_metrics_complex_dashboards_appd/get_rest_url.png "Rest URL from metric browser" %}

If you paste this URL into an internet browser you will see something like this

{% img center /images/ext_metrics_complex_dashboards_appd/rest_results_example.png "Example REST results" %}

The meaning of the parameters in the URL is documented in the AppDynamics documentation. The ones of interest in our case were the *time-range-type* and *duration-in-mins* fields. The *time-range-type* fields is used to get metrics relative to some point in time. For example it can be used to get metrics for the previous 20 minutes or for the hour after 12pm. *duration-in-mins* lets you set the time window size, in this case we would want metrics for 10 minutes and 60 minutes before now, as well as 960 minutes after 5 am in the morning.

### Sending metrics to AppDynamics

Once we can extract the metrics we can sum it up by requesting it for the period we are interested in using URLs such as the following.


*Get calls per minute for information point 'C1' over the last 10 minutes*

	http://controller:8090/controller/rest/applications/ComplexDashboardTest/metric-data?metric-path=Information Points|C1|Calls per Minute&time-range-type=BEFORE_NOW&duration-in-mins=10
 
*Get calls per minute for information point 'C1' for the 960 minutes after 5am this morning*

	http://controller:8090/controller/rest/applications/ComplexDashboardTest/metric-data?metric-path=Information Points|C1|Calls per Minute&time-range-type=AFTER_TIME&start-time=1418148000000&duration-in-mins=960

To send this information to AppDynamics we need the [Standalone Machine agent][appd_machine_agent] to forward it. We created a [Monitoring Extension][appd_monitor_extension] which requested the URLs such as those above replacing the values for each client and changing the time ranges so that we had three metrics for each client. The following is example output of that monitoring extension.

	name=Custom Metrics|Information Points|CLIENT1|Calls per 10 Minutes,value=1
	name=Custom Metrics|Information Points|CLIENT1|Calls per 60 Minutes,value=3
	name=Custom Metrics|Information Points|CLIENT1|Calls per 960 Minutes,value=5
	name=Custom Metrics|Information Points|CLIENT2|Calls per 10 Minutes,value=0
	name=Custom Metrics|Information Points|CLIENT2|Calls per 60 Minutes,value=2
	name=Custom Metrics|Information Points|CLIENT2|Calls per 960 Minutes,value=2543

	... and so on

These new metrics the show up in the controller's metric browser at the following path
	
	Application Infrastructure Performance -> TierName -> Custom Metrics -> Information Points

{% img center /images/ext_metrics_complex_dashboards_appd/new_metrics_in_browser.png "The new metrics displayed in the browser" %}

### Visualising the data

These new metrics can now be tracked by health rules, one for each client. Each rule will raise a WARNING condition if the *Calls per 10 Minutes* is less than 1 but the *Calls per 60 Minutes* is more than 0, and raise a CRITICAL condition if the *Calls per 60 Minutes* is less than 1. Connecting that up to a status light means we have the following

| __Status__    |        __Meaning__                                            |
| --------------| ------------------------------------------                    |
| __GREY__	| N/A								|
| RED           | No activity in the last hour				        |
| YELLOW        | No activity in the last 10 minutes, but some in the last hour |
| GREEN         | Activity in the last 10 minutes                               |

&nbsp;
However there is one piece of information still missing. The grey colour was an indication that there have been no calls the whole day from a particular client, but on the AppDynamics dashboard we only know that there were no calls in the last hour. To fill this gap we overlaid the total number of calls for the day, by that client on the status light. 

The dashboard then looked as follows contain all the information available on the Introscope dashboard.

{% img center /images/ext_metrics_complex_dashboards_appd/client_access_dashboard_final.gif "Completed Dashboard" %}

### In conclusion

The AppDynamics Standalone machine agent provides a very powerful feature, which allows you to send any metrics you might have to the controller, where you can visualize it or react to it. You can even combine your information with that captured by AppDynamics using the REST API, which opens a whole new world of possibilities.


[appd_rest_api]: https://docs.appdynamics.com/display/PRO39/Use+the+AppDynamics+REST+API "AppDynamics REST API"
[appd_info_points]: https://docs.appdynamics.com/display/PRO39/Configure+Code+Metric+Information+Points "AppDynamics Information Points"
[appd_machine_agent]: https://docs.appdynamics.com/display/PRO39/Standalone+Machine+Agent 
[appd_monitor_extension]: https://docs.appdynamics.com/display/PRO39/Add+Metrics+with+a+Monitoring+Extension
[ca_apm]: http://www.ca.com/us/opscenter/ca-application-performance-management.aspx
[pojo]: http://en.wikipedia.org/wiki/Plain_Old_Java_Object

