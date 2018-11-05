---
layout: post
title:  "Find build on specific node for Jenkins"
date:   2018-11-04 18:48:50 +0800
categories: Jenkins
---
I am looking for script like this for a long time, but could not find anyway, so I post here, hopefully it's useful for someone.

```groovy
def busyExecutors = Jenkins.instance.computers.collect {
  		c -> c.executors.findAll { it.isBusy() }
    }.flatten() 
 
 
def nodename = 'n12345'

busyExecutors.each { e ->

  if(e.getOwner().getDisplayName() == nodename){
    
    job = e.getCurrentExecutable().getParent()
    lastbuild = job.getOwnerTask().getLastBuild()
    println "'${lastbuild.getFullDisplayName()}' is currently running on '${nodename}', with building time: ${e.getTimestampString()}" 
  }

  
}
```

```console
'test-test Cookbook #58' is currently running on 'n12345', with building time: 10 min
```