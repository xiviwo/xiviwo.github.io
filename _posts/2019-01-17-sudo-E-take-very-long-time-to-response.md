---
layout: post
title:  "Sudo -E take very long time to response"
date:   2019-01-17 17:30:50 +0800
categories: Sudo Docker
---

So, when trying to execute `sudo -E docker`, it takes for ever to show. 

#### First try, enable docker debug mode

```console
systemctl stop docker.service
dockerd --debug
....
```
It show a lot information, and then stop and never proceed with new info. 
So, it look like it's not caused by docker

#### Second try, debug with strace

```console
strace -t -f sudo -E docker
```

So, generally, it should show info very quick and exit, but this time it stops at for a long time:

```text
09:21:29 open("/var/lib/sss/mc/group", O_RDONLY|O_CLOEXEC) = 6
09:21:29 fstat(6, {st_mode=S_IFREG|0644, st_size=6406312, ...}) = 0
09:21:29 mmap(NULL, 6406312, PROT_READ, MAP_SHARED, 6, 0) = 0x7f827eec2000
09:21:29 fstat(6, {st_mode=S_IFREG|0644, st_size=6406312, ...}) = 0
09:21:29 fstat(6, {st_mode=S_IFREG|0644, st_size=6406312, ...}) = 0
09:21:29 poll([{fd=5, events=POLLIN|POLLOUT}], 1, 300000) = 1 ([{fd=5, revents=POLLOUT}])
09:21:29 poll([{fd=5, events=POLLOUT}], 1, 300000) = 1 ([{fd=5, revents=POLLOUT}])
09:21:29 sendto(5, "\26\0\0\0!\0\0\0\0\0\0\0\0\0\0\0", 16, MSG_NOSIGNAL, NULL, 0) = 16
09:21:29 poll([{fd=5, events=POLLOUT}], 1, 300000) = 1 ([{fd=5, revents=POLLOUT}])
09:21:29 sendto(5, "eidev\0", 6, MSG_NOSIGNAL, NULL, 0) = 6
09:21:29 poll([{fd=5, events=POLLIN}], 1, 300000
```

look at the file of `/var/lib/sss/mc/group`, it's pretty large file for a text file.

```bash
ls  -la /var/lib/sss/mc/group
-rw-r--r-- 1 root root 6406312 Jan 17 09:22 /var/lib/sss/mc/group
```

So, ask google about `/var/lib/sss/mc/group`, so it's System Security Services Daemon (SSSD), generally there is cache issue with sssd.

Let's try to clear cache with 

```bash
systemctl stop sssd
rm -rf /var/lib/sss/db/*
systemctl restart sssd
```

So, it works. `sudo -E ` works as fast as normal.
