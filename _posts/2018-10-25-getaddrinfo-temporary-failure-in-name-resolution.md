---
layout: post
title:  "getaddrinfo: Temporary failure in name resolution"
date:   2018-10-25 10:24:50 +0800
categories: Network
---

I am seeing chef backup failure this morning, which gave me error like:
```console
ERROR: Network Error: Error connecting to https://chef.example.com/organizations/example/cookbooks/example-cookbook/1.1.1 - getaddrinfo: Temporary failure in name resolution
```
It looks odd to me, when I login and try to ping and nslookup `chef.example.com`, it just works fine. 

And,if we use `mtr` to run against target and dns server, it seems to be fine.
But,dns resolution could take up to 200.2(ms), 200 times more than average. And the `StDev` or standard deviation is high, 7.2, it means the latency is inconsistent.

```console
mtr -r -i 0.1 -c 1000 -P 53 dns_server
Start: Thu Oct 25 02:45:00 2018
HOST: x12345                       Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 8.23.121.174               0.0%  1000    0.5   1.0   0.3 200.3   7.2
```

```console
mtr  -r -i 0.1 -c 1000 chef.example.com
Start: Thu Oct 25 02:45:32 2018
HOST: x12345                       Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 7.37.11.24                 0.0%  1000    0.6   0.6   0.4  21.6   0.8
  2.|-- p23irzrv01.example.com     0.0%  1000    0.4   0.4   0.3   8.1   0.5
  3.|-- ???                        100.0  1000    0.0   0.0   0.0   0.0   0.0
  4.|-- x1101321.example.com       0.0%  1000    8.3   8.3   7.9  13.7   0.3
```

to compare with statistic on another node:
```console
mtr -r -i 0.1 -c 1000 -P 53  dns_server
Start: Thu Oct 25 03:19:56 2018
HOST: x30000                      Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 10.17.24.179               0.0%  1000    0.4   0.4   0.2  13.0   1.2
```

if we run the test script on current node and another node, we can easily tell the difference.

***dns_test.rb***

```ruby
#!/usr/bin/ruby
require 'socket'
for i in 0..1000
	puts "---------------#{i}------------------"
	Socket.getaddrinfo("chef.example.com", "https", nil, :STREAM)
end
```
***on current node:***
```console
---------------1000------------------

real	4m45.603s
user	0m0.236s
sys	0m0.208s
```

***on another node:***
```console
---------------1000------------------

real	0m1.554s
user	0m0.171s
sys	0m0.147s
```
So, the dns resolution for current node is unstable and inconsistent, which can time out `getaddrinfo` operation and throw `Temporary failure in name resolution` error. So, the underlying network is most likely the cause of the problem. 