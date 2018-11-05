---
layout: post
title:  "Cheating Sheet of Chef"
date:   2018-11-04 07:24:50 +0800
categories: Chef
---

# List Chef Config of Current Enviroment
you can choose to display different env/config with or without `-c /etc/chef/client.rb`
```console
#=> knife exec -c /etc/chef/client.rb -E 'Chef::Config.keys.each { |c| puts "#{c.to_s} : #{Chef::Config[c.to_sym]}" }'

log_level : warn
log_location : /var/log/chef/client.log
config_file : /etc/chef/client.rb
chef_server_url : https://example.com/organizations/org
client_fork : true
............
```

# Fetch ssl certificate to the correct place
suppose your local knife command uses configuration file `~/.chef/knife.rb`, and you hope to fetch ssl certificate for your `chef-client`, many people tell you that `knife ssl fetch` is fine with that. As a matter of fact, it won't work, it will place cert under `~/.chef` depending your configuration in `~/.chef/knife.rb` . In a word, always include configuration file for specific chef enviroment you hope to work with, in our case, that is `/etc/chef/client.rb` for `chef-client`. So, this will work.


```console
#=> knife ssl fetch -c /etc/chef/client.rb
```

# "VV" is "knife" best friend
If you hope to see what is under the hood for `knife` command, add `-VV`, both upcase.

For example as below, you can see it's actually http `GET` to `https://chef.example.com/organizations/org/nodes/node1` of your chef server behind the scenes. 
```console
#=> knife tag list node1 -VV
INFO: Using configuration from /Users/users/.chef/knife.rb
.............
DEBUG: Signing the request as ei_readonly
.............
DEBUG: Initiating GET to https://chef.example.com/organizations/org/nodes/node1
DEBUG: ---- HTTP Request Header Data: ----
.............
DEBUG: HOST: chef.example.com:443
.............
DEBUG: ---- End HTTP Request Header Data ----
DEBUG: ---- HTTP Status and Header Data: ----
DEBUG: HTTP 1.1 200 OK
.............
{"min_version":"0","max_version":"1","request_version":"1","response_version":"1"}
.............
DEBUG: content-encoding: gzip
DEBUG: ---- End HTTP Status/Header Data ----
.............
DEBUG: HTTP server did not include a Content-Length header in response, cannot identify truncated downloads.
.............
DEBUG: Decompressing gzip response
.............
```
