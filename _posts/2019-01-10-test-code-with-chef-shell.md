---
layout: post
title:  "Test code with chef shell"
date:   2019-01-10 21:38:50 +0800
categories: Chef
---

`chef-shell` is very helpful tool for testing/debuging code. Let's show you with a example.

Say, with a code like below to test:

```ruby
gitlab_script 'reset admin password' do
  action :run
  command <<-SCRIPT
# update root password
user = User.find_by(username: "root")
user.password = 'very_secure_password'
user.password_confirmation = 'very_secure_password'
user.save
SCRIPT
  only_if { some_condition? }
  not_if { another_condition? }
  notifies :run, 'execute[gitlab-ctl restart]', :delayed
end
```

It's would be more handy to test with `chef-shell` than a full-fledged cookbook.

So, `gitlab_script` is customized resource, you still need to spin up a vm with the cookbook having the definition of the resource. 

Then, run `chef-shell -z` to load all the dependencies and modules, it could take a while.

```console
chef-shell -z
loading configuration: /etc/chef/client.rb
Session type: client
.resolving cookbooks for run list: ["test-cookbook@1.0.0"]
..................Synchronizing Cookbooks:
  - build-essential (8.0.4)
  - apt (4.0.1)
  - chef-client (10.0.0)
....done.

This is the chef-shell.
 Chef Version: 13.6.4
 https://www.chef.io/
 https://docs.chef.io/

run `help' for help, `exit' or ^D to quit.

Ohai2u root@12345!
chef (13.6.4)>
```
Ok, chef shell is ready, now switch to `recipe_mode`.

```console
chef (13.6.4)> recipe_mode
```

Paste the code there and run `run_chef`, there you go 

Oops, it complains `NoMethodError: undefined method some_condition? for Custom resource`.

So, we forget to define the `some_condition?` method, let's try this

```ruby
module Gitlab
  module Helpers
    def some_condition?()
        puts 'do something'
        true
    end   
    def another_condition?()
        puts 'do anther thing'
        false
    end 
  end
end
```

We have to leave `recipe_mode` to make it work, also the module needs to be included in recipe with `Chef::Recipe.send(:include, Gitlab::Helpers)` and enter `recipe_mode` again to test

```console
chef (13.6.4)> module Gitlab
chef ?>   module Helpers
chef ?>     include Gitlab::Psql
chef ?>     def some_condition?()
chef ?>         puts 'do something'
chef ?>         true
chef ?>     end   
chef ?>     def another_condition?()
chef ?>         puts 'do anther thing'
chef ?>         false
chef ?>     end 
chef ?>   end
chef ?> end
 => :another_condition? 
chef (13.6.4)> Chef::Recipe.send(:include, Gitlab::Helpers)
 => Chef::Recipe 
chef (13.6.4)> recipe_mode
chef:recipe (13.6.4)> another_condition?
do anther thing
 => false 
chef:recipe (13.6.4)> some_condition?
do something
 => true 
```
So, we are happy to execute the first code block again, but Ooops again...

it still complains `undefined method some_condition? for Custom resource`

Why ? it's just working fine in the `recipe_mode` of `chef shell` ?

So, note `Custom resource` of the error, it means the method is not defined in the namespace of `chef resource`. 

It works for `recipe_mode` because we include the module in the `chef recipe` namespace with `Chef::Recipe.send(:include, Gitlab::Helpers)`

The answer is to include it into `chef resource` with `Chef::Resource.send(:include, Gitlab::Helpers)`


```console
chef (13.6.4)> Chef::Resource.send(:include, Gitlab::Helpers)
 => Chef::Resource 
chef (13.6.4)> recipe_mode
chef:recipe (13.6.4)> gitlab_script 'reset admin password' do
chef:recipe >   action :run
chef:recipe ?>   command <<-SCRIPT
chef:recipe"> # update root password
chef:recipe"> user = User.find_by(username: "root")
chef:recipe"> user.password = 'very_secure_password'
chef:recipe"> user.password_confirmation = 'very_secure_password'
chef:recipe"> user.save
chef:recipe"> SCRIPT
chef:recipe ?>   only_if { some_condition? }
chef:recipe ?>   not_if { another_condition? }
chef:recipe ?>   notifies :run, 'execute[gitlab-ctl restart]', :delayed
chef:recipe ?> end
chef:recipe (13.6.4)> run_chef
```

Now, it's happy to work without error