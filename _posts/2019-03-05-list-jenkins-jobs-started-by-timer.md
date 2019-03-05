---
layout: post
title:  "List Jenkins Jobs Started by Timer"
date:   2019-03-05 18:03:50 +0800
categories: Jenkin
---

Sometimes, we hope to find out those jobs started by timer , and get info from it

Here, it's a simple example


```groovy
def busyExecutors = Jenkins.instance.computers.collect {
                                  c -> c.executors.findAll { it.isBusy() }
                                }
                                .flatten()
 
 
println "Build started by timer list"
busyExecutors.each { e ->

  if(e.isBusy()){
    job = e.getCurrentExecutable().getParent()
    build = job.getOwnerTask().getLastBuild()
    def action = build.getAction(hudson.model.CauseAction.class)
    // get the list of causes
    def found = false
    for (cause in action.getCauses()) {
      if(cause.getShortDescription() == 'Started by timer'){
        found = true
        break;
      }
    }

    if(found){
      println '[Time elapsed]: ' + e.getTimestampString()
      println job.getFullDisplayName() +  ' is timer build, currently run on ' + e.getOwner().getDisplayName()
     
    }

    println '-----------------------------'
  }
}

```

Sample output:

```console
Build started by timer list
[Time elapsed]: 4 min 10 sec
my-user Cookbook #349 (0) is timer build, currently run on master
-----------------------------
[Time elapsed]: 3 min 9 sec
my-build Cookbook #155 is timer build, currently run on master
-----------------------------
[Time elapsed]: 16 min
my-database Cookbook #158 (0) is timer build, currently run on master
-----------------------------
[Time elapsed]: 1 min 9 sec
my-apache Cookbook #261 (static analysis) is timer build, currently run on n10203
```