---
layout: post
title:  "Stop Stuck Jobs in Jenkins"
date:   2018-10-11 21:19:50 +0800
categories: Jenkins
---
If you found some jenkins jobs are stuck in hours without exit, you probally have a try with following script:

This script trys to abort those jobs exceed one hour's execution.

```groovy

Jenkins.instance.getItems().collect { 

	job-> job.builds.findAll { it.getResult().equals(null) }

}.flatten().each { 
    it -> 
    if((System.currentTimeMillis() - it.getStartTimeInMillis() ) > 3600000){
        println it.getFullDisplayName() + " - " + it.getDurationString() 
        it.finish(hudson.model.Result.ABORTED,new java.io.IOException("Aborting build"))
    }
}
```

PS: It looks like previous command only abort building instead of `quit` the build, it will leave the execute to be there FOREVER!

So, following maybe more suitiable for practical need.

```groovy

Jenkins jenkins = Jenkins.instance
for (Computer node in jenkins.computers) {
  // Make sure slave is online
  println "Checking " + node.getDisplayName()
  if (!node.isOnline()) {
    println "Node " + node.getDisplayName() + " is currently offline - skipping check"
  } else {

    for (Executor ex : node.getExecutors() ){
      // if executor is running and take more than one hours, we believe it's dead
      if(ex.isBusy() && ex.getElapsedTime() > 3600000){
        println '[Time Used]' + ex.getTimestampString()
        // try to play nicely
        println ex.getDisplayName() +  ' is DEAD, try to stop on ' + node.getDisplayName()
        ex.doStop()
      }

    }

  }

}
```