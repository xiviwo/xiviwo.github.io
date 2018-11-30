---
layout: post
title:  "Use chef-provisioning-vsphere to provision multiple machines"
date:   2018-11-07 14:33:50 +0800
categories: Chef 
---

In general, we depend on `.kitchen.yml` to spin up machine to test `chef` cookbook, it's just ok in most case.

But, what if we need interplay test between nodes, like I need one machine acting as `master` and one machine as `slave`. Yes, we can have faked fixture coming with json file in the `test/nodes` directory.

But, I hope to do something real, which can't be simulated with faked fixture.

I think `chef-provisioning-vsphere` can be one options of this.


Basically, firstly we need to setup configuration to your vsphere server and then just create a few of `machine` resources, and that is it.

For example

***vsphere_setup.rb***
```ruby

# need to install the gem of it, in order for the following to work
 chef_gem 'chef-provisioning-vsphere' do
  action :install
  compile_time true
end

require 'chef/provisioning'
require 'chef/provisioning/vsphere_driver'

# setup configure with your vsphere
with_vsphere_driver host: 'vsphere.example.com',
                    insecure: true,
                    user:     'administrator@vsphere.local',
                    password: 'p@ssw0rd'

# basic configuration with your machine to build
with_machine_options bootstrap_options: {
  num_cpus: 4,
  memory_mb: 4096,
  network_name: ['you-network'],
  datacenter: 'your beautiful Datacenter',
  resource_pool: 'your beautiful Pool',
  template_name: 'your beautiful template',
  template_folder: 'Templates',
  customization_spec: {
    ipsettings: {
      dnsSuffixList: ['.local']
    },
    domain: 'example.com'
  },
  ssh: {
    user: 'vagrant',
    keys: ['.ssh/vagrant_rsa'],
    paranoid: false,
    port: '22'
  }
},
 convergence_options: {
   ssl_verify_mode: :verify_none,
   chef_config: 'log_level :info',
   client_rb_path: '/etc/chef_provision/client.rb',
   client_pem_path: '/etc/chef_provision/client.pem',
 },
 sudo: true,
 ssh_options: { # a list of options to Net::SSH.start
   auth_methods: ['publickey'], # DEFAULT
   # :keys_only => true, # DEFAULT
   # :host_key_alias => "#{instance.id}.AWS", # DEFAULT
   # :key_data => nil, # use key from ssh-agent instead of a local file; remember to ssh-add your keys!
   forward_agent: true, # you may want your ssh-agent to be available on your provisioned machines
   never_forward_localhost: false, # This will, if set, disable SSH forwarding if it does not work/make sense in your envirnoment
   remote_forwards: [
     # Give remote host access to squid proxy on provisioning node
     { remote_port: 8889, local_host: 'localhost', local_port: 1 },
     # Give remote host access to private git server
     # {:remote_port => 2222, :local_host => 'git.example.com', :local_port => 22,},
   ],
   # You can send net-ssh log info to the Chef::Log if you are having
   # trouble with ssh.
   logger: Chef::Log,
   # If you use :logger => Chef::Log and :verbose then your :verbose setting
   # will override the global Chef::Config. Probably don't want to do this:
   #:verbose => :warn,
 }


```

Then, use `machine` resource to do the real work. Here, we spin up two machine, with their specific recipes, and attributes can also be assigned here.

***provision.rb***
```ruby

machine 'jenkins-node' do
  recipe 'jenkins::client'
  tag 'Jenkins-slave'
end

machine 'jenkins-master' do
  recipe 'jenkins::master'
  recipe 'jenkins-test::jenkins_test'
  attribute %w(jenkins master jenkins_prefix), '/jenkins_test'
  attribute %w(jenkins master jenkins_baseurl), 'http://localhost:8080'
  tag 'Jenkins_master'
end
```

Note that chef-zero does not support berkshelf style cookbook dependency resolution. We need to perform a `berks vendor` to pull down the dependencies first.

But, please **DO NOT** place berks cookbook under the same directory as cookbook you are going to test with, because everytime `berks vendor` trys to copy everything under your cookbook into `./berks-cookbooks`, which will make `berks-cookbooks` growing more and more bigger! Something like `berks-cookbooks/jenkins/berks-cookbooks/jenkins/berks-cookbooks/jekins/berks-cookbooks/....`, look exactly like a dead lock....

We should do this:
```console
#=> berks vendor ../berks-cookbooks
```

Now, come to the real provision:

```console
#=> chef-client -z provision/vsphere_setup.rb provision/provision.rb --listen --config-option cookbook_path=$(pwd)/../berks-cookbooks/ 
```

You should be seesing something like this:
```console
Starting Chef Client, version 13.6.4
resolving cookbooks for run list: []
Synchronizing Cookbooks:
Installing Cookbook Gems:
Compiling Cookbooks...
Recipe: @recipe_files::/Users/user/chef/cookbooks/jenkins/provision/vsphere_setup.rb
  * chef_gem[chef-provisioning-vsphere] action install (up to date)
[2018-11-07T11:13:51+08:00] WARN: Node test_node has an empty run list.
[2018-11-07T11:13:51+08:00] WARN: Node test_node has an empty run list.
  Converging 2 resources
  * chef_gem[chef-provisioning-vsphere] action install (up to date)
Recipe: @recipe_files::/Users/user/chef/cookbooks/jenkins/provision/provision.rb
  * machine[jenkins-node] action converge[2018-11-07T11:13:51+08:00] WARN: Checking to see if {"driver_url"=>"vsphere://example.com/sdk?use_ssl=true&insecure=true", "driver_version"=>"1.2.2.3", "server_id"=>"50108654-aae2-789a-d33b-099d7a0b4eb7", "is_windows"=>false, "allocated_at"=>"2018-11-07 01:13:39 UTC", "ipaddress"=>"......"} has been created...
    - creating jenkins-node on  (example.com)
    - finding networks...
    - network: yournetwork
Cloning VM...40%...52%...62%...72%...91%...Done!

    - jenkins-node created
     id: 50109a86-d71b-7c63-4ec6-540dd4a9f088 on  (example.com)
    - update node jenkins-node at chefzero://localhost:8889
    -   update normal.chef_provisioning.reference.allocated_at from "2018-11-07 01:13:39 UTC" to "2018-11-07 03:16:47 UTC"
    -   update normal.chef_provisioning.reference.ipaddress from "........" to nil
    - waiting for jenkins-node to be ready
..........................    - 
    jenkins-node is now ready.
    - IP address obtained: ..........
    - update node jenkins-node at chefzero://localhost:8889
    -   update normal.chef_provisioning.reference.ipaddress from nil to "......"
```

Ok, after a while you should be able to have two nodes to play with, after the test, you can even deprovision them with `machine` resource again

***deprovision.rb***
```ruby

query = 'recipe:jenkins*client OR recipe:jenkins*master'

names = search(:node, query).map(&:name)
log "Deprovisioning #{names}"

machine_batch do
  machines names
  action :destroy
end
```

Use similar command:
```console
#=> chef-client -z provision/vsphere_setup.rb provision/deprovision.rb --listen
```

Your probally will see something like this:
```console
Starting Chef Client, version 13.6.4
resolving cookbooks for run list: []
Synchronizing Cookbooks:
Installing Cookbook Gems:
Compiling Cookbooks...
Recipe: @recipe_files::/Users/user/chef/cookbooks/jenkins/provision/vsphere_setup.rb
  * chef_gem[chef-provisioning-vsphere] action install (up to date)
  Converging 3 resources
  * chef_gem[chef-provisioning-vsphere] action install (up to date)
Recipe: @recipe_files::/Users/user/chef/cookbooks/jenkins/provision/deprovision.rb
  * log[Deprovisioning ["jenkins-node", "master"]] action write
  
  * machine_batch[default] action destroy
    - [master] Delete VM [vm/master]
    - [jenkins-node] Delete VM [vm/jenkins-node]
    - [jenkins-node] delete node jenkins-node at chefzero://localhost:8889
    - [master] delete node master at chefzero://localhost:8889
    - [jenkins-node] delete client jenkins-node at clients
    - [master] delete client master at clients
```