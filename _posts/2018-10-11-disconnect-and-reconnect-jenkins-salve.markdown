---
layout: post
title:  "Disconnect and Reconnect Jenkins Slave"
date:   2018-10-11 21:29:50 +0800
categories: Jenkins
---
Disconnect slave from master for maintenance.

{% highlight java %}

def list = ['xxxx','xxxx']

jenkins = Hudson.instance
for (slave in jenkins.slaves) {
  def computer = slave.computer
  if(list.contains(computer.name)){
    println "Computer ${computer.name} in list"
    computer.disconnect(new hudson.slaves.OfflineCause.ByCLI("Maintenance"))
    println('computer.getOfflineCause: ' + slave.getComputer().getOfflineCause());
    println('computer.offline: ' + computer.offline); 
  }
}
{% endhighlight %}

Reconnect slave to master

{% highlight java %}

def list = ['xxxx','xxxx']
 
jenkins = Hudson.instance
for (slave in jenkins.slaves) {
  def computer = slave.computer
  if(list.contains(computer.name)){
    println "Computer ${computer.name} in list"
    println("bringing slave back online")	
    computer.connect(true)  
    println('computer.getConnectTime: ' + new Date(computer.getConnectTime()));
    println('computer.offline: ' + computer.offline); 
  }
}

{% endhighlight %}