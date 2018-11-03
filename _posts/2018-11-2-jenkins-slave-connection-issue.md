---
layout: post
title:  "Jenkins Slave Connection Issue"
date:   2018-11-02 10:23:50 +0800
categories: Jenkins
---

I was paged by monitoring team yesterday, claiming `paging space issue`.

When I login the node and check, I found there are four `VBoxHeadless` processes there.
It's `Virtualbox` vm running. 

Thing is not right, since one node is supposed to run only one build at a time, which is enforced by our policy. 

And, I check the jenkins console, I found the job building matches one of the `VBoxHeadless` process there. So, I believe the three of the rest are left hung without clean stop. 

Apparently, the system resource is consumed too much by jenkins building as well as paging space, which is causing the `paging space issue` alert.

```console
date;for file in /proc/*/status ; do awk '/VmSwap|Name|^Pid/{printf $2 " " $3}END{ print ""}' $file; done | sort -k 3 -n -r | head -n 12
Thu Nov  1 11:01:35 UTC 2018
kitchen 26005 266400 kB
java 12592 116172 kB
java 22462 100812 kB
VBoxHeadless 12993 46400 kB
VBoxHeadless 12987 44976 kB
VBoxHeadless 2867 40508 kB
VBoxHeadless 24946 37972 kB
```


It happened before, so let's cleanup the hung vm.

Found the IDs of running vms
```shell
VBoxManage list runningvms

"default-RHEL_default_1541117567388_87036" {12339317-025d-46d0-80b0-ba5dcf0da64c}
"default-RHEL_default_1541123403946_99283" {81d1f63b-39ef-4c02-8d45-ca85d6160028}
"default-RHEL_default_1541125255909_20451" {bcc32a86-c38b-40cf-9890-bc197d920870}
"default-RHEL_default_1541126348591_77560" {e8a3269f-f611-472f-8826-d891bf2d21df}
```


Exclude the one actually running, stop the rest with
```shell
clean_virtualbox() {
    VBoxManage controlvm $1 poweroff 
    VBoxManage unregistervm $1 --delete
}
clean_virtualbox bcc32a86-c38b-40cf-9890-bc197d920870
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
```

OK, solved, close ticket. Oh, no, not again. Yes, it happens again after noon.

Then, I notice many jenkins slave goes offline unexpected, in the middle of a build.

And, I check the log of slave, it reads:
```console
Connection was broken

java.util.concurrent.TimeoutException: Ping started at 1541121552889 hasn't completed by 1541121792890
    at hudson.remoting.PingThread.ping(PingThread.java:134)
    at hudson.remoting.PingThread.run(PingThread.java:90)
```

It looks unusual at first glance, but let's it back online, and monitor.
```groovy
def list = ['x21044']
 
jenkins = Hudson.instance
for (slave in jenkins.slaves) {
  def computer = slave.computer
  if(list.contains(computer.name)){
    println "Computer ${computer.name} is offline"
    println("bringing slave back online")   
    computer.connect(true)  
    computer.setTemporarilyOffline(false, null)
    println('computer.getConnectTime: ' + new Date(computer.getConnectTime()));
    println('computer.getLog: ' + computer.getLog());
  }
}
```

Then, it's back online and start to build again happily.
```console
[SSH] Starting slave process: cd "/home/jenkins" && java  -jar slave.jar
<===[JENKINS REMOTING CAPACITY]===>channel started
Remoting version: 3.14
This is a Unix agent
Evacuated stdout
Agent successfully connected and online
```

But, after a while, all in a suddent, the agent goes offline in the middle of build, yes, again, with same error. 

So, it comes more clear that, the hung vms are caused by connection interruption, the build is not cleaned up gracefully, so that I am seeing a lot of hung vms there.

I notice another interesting thing, connection issue only happen to jenkins slaves on one available zone, slaves on another zone are all just fine.

In regards to broken connection and the interesting symptom, the first thing coming to my mind is network.

ok, let's check networking.
```shell
for host in slave1 slave2 slave3 slave4 
do
    echo -----------------testing $host-------------------------
    mtr -r -c 100 -P 22 $host
done
```

The result shows consistently, packet loss is happening in one avaibility zone and not in other zone.
```shell
Start: Fri Nov  1 03:27:01 2018
HOST: jenkinsmaster              Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 90.4.24.35               0.0%   100    0.9   1.0   0.5  10.9   1.3
  2.|-- p9eirzrv01.example.com   0.0%   100    0.8   1.3   0.5  12.2   2.1
  3.|-- ???                      100.0   100    0.0   0.0   0.0   0.0   0.0
  4.|-- slave1.example.com       5.0%   100   21.4  22.0  21.3  31.0   1.5
```

So, I have to blame network again.