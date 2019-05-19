---
layout: post
title:  "Cheating Sheet of Chef"
date:   2018-11-04 07:24:50 +0800
categories: Chef
---

## List Chef Config of Current Enviroment
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
Or simply
```console
#=> knife exec -c /etc/chef/client.rb -E 'puts Chef::Config.inspect'
```

## Fetch ssl certificate to the correct place
suppose your local knife command uses configuration file `~/.chef/knife.rb`, and you hope to fetch ssl certificate for your `chef-client`, many people tell you that `knife ssl fetch` is fine with that. As a matter of fact, it won't work, it will place cert under `~/.chef` depending your configuration in `~/.chef/knife.rb` . In a word, always include configuration file for specific chef enviroment you hope to work with, in our case, that is `/etc/chef/client.rb` for `chef-client`. So, this will work.


```console
#=> knife ssl fetch -c /etc/chef/client.rb
```

## "VV" is "knife" best friend
If you hope to see what is under the hood for `knife` command, add `-VV`, both upcase.

For example as below, you can see it's actually http `GET` to `https://chef.example.com/organizations/org/nodes/node1` of your chef server behind the scenes. 
```console
#=> knife tag list node1 -VV
INFO: Using configuration from /Users/users/.chef/knife.rb
.............
DEBUG: Signing the request as test_node
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

## Enable "DEBUG" logging for test-kitchen

Just update `.kitchen.yml` and add `log_level: debug` below `provisioner` as below.
```yaml
provisioner:
  log_level: debug
```

Or login the VM and run `chef-client` directly
```
sudo -E /opt/chef/bin/chef-client --local-mode --config /tmp/kitchen/client.rb --log_level debug --force-formatter --no-color --json-attributes /tmp/kitchen/dna.json --chef-zero-port 8889
```
Or simple one

```
cd /tmp/kitchen && chef-client -z -j dna.json
```

## Enable "DEBUG" logging for chefspec

Add lines below to `spec_helper.rb`
```ruby
# spec_helper.rb
RSpec.configure do |config|
  config.log_level = :debug
end
```
## Get Path of your chef run command

get the path of your chef run, not the one system-wide

```console
#=> chef exec "which berks"
/opt/chefdk/bin/berks
#=> which berks
/opt/chefdk/embedded/bin/berks
```

## Run chef-shell in test-kitchen
before that, login the vm with `kitchen login <suites name>`

without run-list
```
cd /tmp/kitchen && chef-shell -c client.rb -j dna.json
```

with run-list
```
cd /tmp/kitchen && chef-shell -c client.rb -j dna.json -s
```

add debug output
```
cd /tmp/kitchen && chef-shell -c client.rb -j dna.json -l debug
```

## Run knife command in test-kitchen

Basically, adding `-z` is all about that 

```
cd /tmp/kitchen 
knife node list -z
knife client list -z
knife cookbook list -z
knife status -z
knife search node "*:*" -z -a ipaddress
knife tag list $(knife node list -z) -z

```

## Search node with specified attribute

Say, to find nodes with `sshd` cookbooks of version `2.0.0`

```
knife search node  "cookbooks_sshd_version:2.0.0" -a name
2 items found

gitlab:
  name: gitlab

gitlab-test-RHEL7:
  name: gitlab-test-RHEL7
```

## Showing node (nested) attributes

showing node's name
```
knife node show gitlab -a name
gitlab:
  name: gitlab
```

showing node's ip adress
```
knife node show gitlab -a ipaddress
gitlab:
  ipaddress: 10.0.2.15
```

showing node's all recipes
```
knife node show gitlab -a recipes
gitlab:
  recipes:
    build-essential::default
    yum::default
    logrotate::default
    chef-sugar::default
```

showing node's plaform
```
knife node show gitlab -a platform
gitlab:
  platform: redhat
```

showing node's all cookbooks
```
knife node show gitlab cookbooks
gitlab:
  cookbooks:
    build-essential:
      version: 8.2.1
    logrotate:
      version: 2.2.0
    mingw:
      version: 2.1.0
    seven_zip:
      version: 3.1.0
    ssh:
      version: 0.10.24
    sshd:
      version: 2.0.0
```

showing node's specific cookbook version
```
knife node show gitlab -a cookbooks.ssh.version
gitlab:
  cookbooks.ssh.version: 0.10.24
```

## --skip-cookbook-sync 

say, if you find some bug or problem on specific node, you hope to test that fix out before actually commit the code to repository, `--skip-cookbook-sync` is very good choice of that

#### update the change in place.

`chef-client` generally caches cookbook at `/var/chef/cache/cookbooks`. Say the problematic recipes is located at `/var/chef/cache/cookbooks/mycookbook/recipes/default.rb`, edit that file with the fix via any text editor your prefer, like vi, nano, etc.

#### converge with --skip-cookbook-sync

```
/usr/bin/chef-client --skip-cookbook-sync
```
now, you can verify the fix

## Upload community cookbook to chef server

```
knife cookbook site download jenkins 6.2.1
Downloading jenkins from Supermarket at version 6.2.1 to /tmp/kitchen/jenkins-6.2.1.tar.gz
Cookbook saved: /tmp/kitchen/jenkins-6.2.1.tar.gz
mkdir -v cookbooks/
tar xf jenkins-6.2.1.tar.gz -C cookbooks/
knife cookbook upload jenkins -o cookbooks --freeze
Uploading jenkins        [6.2.1]
```
you must handle dependency one by one similarly.
