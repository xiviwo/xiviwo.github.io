---
layout: post
title:  "Abort Stuck Jobs in Jenkins"
date:   2018-10-11 21:19:50 +0800
categories: Jenkins
---
If you found some jenkins jobs are stuck in hours without exit, you probally have a try with following script:

This script trys to abort those jobs exceed one hour's execution.

{% highlight groovy %}

Jenkins.instance.getItems().collect { 

	job-> job.builds.findAll { it.getResult().equals(null) }

}.flatten().each { 
    it -> 
    if((System.currentTimeMillis() - it.getStartTimeInMillis() ) > 3600000){
        println it.getFullDisplayName() + " - " + it.getDurationString() 
        it.finish(hudson.model.Result.ABORTED,new java.io.IOException("Aborting build"))
    }
}
{% endhighlight %}
