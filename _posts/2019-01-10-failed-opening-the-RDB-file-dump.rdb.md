---
layout: post
title:  "Failed opening the RDB file dump.rdb"
date:   2019-01-10 09:44:50 +0800
categories: Gitlab Redis
---

Today, after restart gitlab with `gitlab-ctl start`, `redis` does not come up.

```console
#=> sudo gitlab-ctl status
run: alertmanager: (pid 29241) 226s; run: log: (pid 2038) 78273s
run: gitaly: (pid 29257) 226s; run: log: (pid 1836) 78277s
run: gitlab-monitor: (pid 29270) 225s; run: log: (pid 1876) 78276s
run: gitlab-workhorse: (pid 29288) 225s; run: log: (pid 1813) 78277s
run: logrotate: (pid 29304) 224s; run: log: (pid 17042) 76838s
run: nginx: (pid 29321) 224s; run: log: (pid 1814) 78277s
run: node-exporter: (pid 29333) 223s; run: log: (pid 1861) 78276s
run: prometheus: (pid 29416) 223s; run: log: (pid 2007) 78274s
run: redis: (pid 15776) 64867s, got TERM; down: log: 1s, normally up, want up
```

Look into the log, it reports

```console
#=>sudo gitlab-ctl tail redis
2019-01-10_01:22:39.12535 31515:M 10 Jan 01:22:39.125 # Background saving error
2019-01-10_01:22:45.04017 31515:M 10 Jan 01:22:45.039 * 1 changes in 900 seconds. Saving...
2019-01-10_01:22:45.04020 31515:M 10 Jan 01:22:45.040 * Background saving started by pid 3636
2019-01-10_01:22:45.04072 3636:C 10 Jan 01:22:45.040 # Failed opening the RDB file dump.rdb (in server root dir unknown) for saving: No such file or directory
```

So, there are many posts telling what could be the problem is, like permission, redis configuration. But, all doesn't work for me.

And, finally, it turns out there are a few redis process are running.

```console
#=> ps ax | grep redis-server
 6405 pts/0    S+     0:00 grep --color=auto redis-server
15776 ?        Ssl    4:33 /opt/gitlab/embedded/bin/redis-server 127.0.0.1:0
25809 ?        Ssl    1:08 /opt/gitlab/embedded/bin/redis-server 127.0.0.1:0
31515 ?        Ssl    1:27 /opt/gitlab/embedded/bin/redis-server 127.0.0.1:0
```

After closing those hung process, everything is fine. hopefully, it's helpful for some one.