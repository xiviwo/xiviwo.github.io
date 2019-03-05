---
layout: post
title:  "How to converge chef cookbook locally"
date:   2019-03-04 21:45:50 +0800
categories: Chef
---

Say we want to develop chef cookbook locally or we only have very few nodes to appy chef cookbook, so we don't need dedicted chef server. So how ? 

Basically, there is nothing magical here but local mode of `chef-client`, that is:

```console
sudo chef-client -z 
```

The magic lies in that where do you get the dependent cookbook or cookbook itself ? 

The answer is [Berkshelf][berkshelf], which still doesn't look much magic though.

Yes, it look pretty easy with open cookbook repository like Chef [supermarket][supermarket]

We are assuming `chefdk` or `chef-client` is installed on your machine for following steps to happen.

### Open Cookbook Repository

Let's take a very simple java installation cookbook for example.

#### a. Create Berksfile file for Berkshelf to download cookbook

```bash
cat > Berksfile <<'EOP'
source 'https://supermarket.chef.io'

cookbook 'java'
EOP
```
basically, it tells `Berkshelf` to download `java` cookbook along with its dependency from Chef supermarket, https://supermarket.chef.io

#### b. Actually download cookbook

```bash
berks vendor
Resolving cookbook dependencies...
Fetching cookbook index from https://supermarket.chef.io...
Using homebrew (5.0.8)
Installing java (3.2.1)
Using windows (5.2.4)
Vendoring homebrew (5.0.8) to cookbooks/homebrew
Vendoring java (3.2.1) to cookbooks/java
Vendoring windows (5.2.4) to cookbooks/windows
```
So, it places all cookbooks in directory called `cookbooks` as the command denotes.

```bash
#=> ls -lh cookbooks/
total 12K
drwxr-xr-x 6 root root 4.0K Mar  4 13:23 homebrew
drwxr-xr-x 7 root root 4.0K Mar  4 13:23 java
drwxr-xr-x 6 root root 4.0K Mar  4 13:23 windows
```

#### c. Converge locally 

```bash
sudo chef-client -z -o java
[2019-03-04T13:24:33+00:00] WARN: No config file found or specified on command line, using command line options.
Starting Chef Client, version 13.6.4
[2019-03-04T13:24:40+00:00] WARN: Run List override has been provided.
[2019-03-04T13:24:40+00:00] WARN: Run List override has been provided.
[2019-03-04T13:24:40+00:00] WARN: Original Run List: []
[2019-03-04T13:24:40+00:00] WARN: Original Run List: []
[2019-03-04T13:24:40+00:00] WARN: Overridden Run List: [recipe[java]]
[2019-03-04T13:24:40+00:00] WARN: Overridden Run List: [recipe[java]]
resolving cookbooks for run list: ["java"]
Synchronizing Cookbooks:
  - java (3.2.1)
  - windows (5.2.4)
  - homebrew (5.0.8)
Installing Cookbook Gems:
Compiling Cookbooks...
Converging 5 resources
Recipe: java::notify
  * log[jdk-version-changed] action nothing (skipped due to action :nothing)
Recipe: java::openjdk
  * yum_package[java-1.6.0-openjdk, java-1.6.0-openjdk-devel] action install

    - install version 1.6.0.41-1.13.13.1.el7_3 of package java-1.6.0-openjdk
    - install version 1.6.0.41-1.13.13.1.el7_3 of package java-1.6.0-openjdk-devel
Recipe: java::notify
  * log[jdk-version-changed] action write
  
Recipe: java::openjdk
  * java_alternatives[set-java-alternatives] action set
    - Add alternative for appletviewer
    .........
Recipe: java::set_java_home
  * directory[/etc/profile.d] action create (up to date)
  * template[/etc/profile.d/jdk.sh] action create
    - update content in file /etc/profile.d/jdk.sh from 7f8009 to 778ff5
    --- /etc/profile.d/jdk.sh	2019-02-15 09:16:35.837102552 +0000
    +++ /etc/profile.d/.chef-jdk20190304-1795-1ra936a.sh	2019-03-04 13:29:31.564830872 +0000
    @@ -1,2 +1,2 @@
    +export JAVA_HOME=/usr/lib/jvm/java-1.6.0
[2019-03-04T13:29:31+00:00] WARN: Skipping final node save because override_runlist was given
[2019-03-04T13:29:31+00:00] WARN: Skipping final node save because override_runlist was given

Running handlers:
Running handlers complete
Chef Client finished, 4/6 resources updated in 04 minutes 57 seconds
```
Nice, it looks pretty clean and nicely to work with, but what if you have one private cookbook repository ? 

### Private Cookbook Repository

#### a. Update Berksfile

Say, my cookbook is from gitlab, and dependent cookbooks are from chef_server
```bash
#=> cat > Berksfile <<'EOP'
source :chef_server

cookbook 'mycookbook', git: "https://gitlab.example.com/cookbooks/mycookbook.git"
EOP
```
Run again, oooops...

```bash
#=> berks vendor
Resolving cookbook dependencies...
Fetching 'mycookbook' from https://gitlab.example.com/cookbooks/mycookbook.git (at master)
Fetching cookbook index from chef_server: https://localhost:443...
[2019-03-04T14:41:31+00:00] WARN: Failed to read the private key /etc/chef/client.pem: #<Errno::ENOENT: No such file or directory @ rb_sysopen - /etc/chef/client.pem>
/opt/chefdk/embedded/lib/ruby/gems/2.4.0/gems/chef-13.6.4/lib/chef/http/authenticator.rb:95:in `rescue in load_signing_key': I cannot read /etc/chef/client.pem, which you told me to use to sign requests! (Chef::Exceptions::PrivateKeyMissing)
```

So, it need authentication with `chef_server`, so let's add a `client.pem` of some valid chef `client` and a `knife.rb`.



#### b. Create client.pem and knife.rb



***Note: we use `knife.rb` instead of `client.rb`, because A `client.rb` file is default configuration for the `chef-client` or `node` machine; A `knife.rb` file is default configuration for a chef `client` machine, some like `knife` command.  Sound pretty confusing? right, but if are aware of the difference between chef `node` and `client`, it will make more sense to you.***


As `/etc/chef/` is default directory for configuration file of `chef-client`, in order not to interfere with current `chef-client` run, I choose a different directory to test with 

```bash
#=> mkdir -pv /tmp/chef
#=> cat > /tmp/chef/client.pem <<'EOP'
-----BEGIN RSA PRIVATE KEY-----
.............
cUBiob2YrYYBgav5PT1Ubyf4m3kKnj73u67r6e5bFcAnM2PSB4E=
-----END RSA PRIVATE KEY-----

EOP
```

So let's create `/tmp/chef/knife.rb`. 


```bash
#=> cat > /tmp/chef/knife.rb <<'EOP'
 log_level        :info
 log_location     STDOUT
 node_name        'one_valid_client'
 client_key       '/tmp/chef/client.pem'
 chef_server_url  'https://chef.example.com/organizations/test_org'
 chef_repo_path   Dir.pwd
 ssl_verify_mode  :verify_none
EOP
```

Now, it works

```bash
#=> berks vendor
Resolving cookbook dependencies...
Fetching 'mycookbook' from https://gitlab.example.com/cookbooks/mycookbook.git  (at master)
Fetching cookbook index from chef_server: https://chef.example.com/organizations/test_org...
Using build-essential (8.0.4)
.......
Using mycookbook (0.0.0) from https://gitlab.example.com/cookbooks/mycookbook.git (at master)
.......
Vendoring windows (3.4.1) to /tmp/chef/berks-cookbooks/windows
Vendoring yum (5.0.1) to /tmp/chef/berks-cookbooks/yum
```

So, come to the exciting part, let's converge with `chef-client -z`

```bash
#=> chef-client -z -o 'mycookbook'
[2019-03-05T01:49:23+00:00] INFO: Started chef-zero at chefzero://localhost:1 with repository at /tmp/chef
  One version per cookbook

[2019-03-05T01:49:23+00:00] INFO: Forking chef instance to converge...
Starting Chef Client, version 13.6.4
......
[2019-03-05T01:49:26+00:00] WARN: Overridden Run List: [recipe[mycookbook]]
[2019-03-05T01:49:26+00:00] INFO: Run List is [recipe[mycookbook]]
[2019-03-05T01:49:26+00:00] INFO: Run List expands to [mycookbook]
[2019-03-05T01:49:26+00:00] INFO: Starting Chef Run for one_valid_client
....
resolving cookbooks for run list: ["mycookbook"]

================================================================================
Error Resolving Cookbooks for Run List:
================================================================================

Missing Cookbooks:
------------------
No such cookbook: mycookbook

Expanded Run List:
------------------
* mycookbook
```

Hmmmm, look like it's not aware of where the cookbooks are located

#### c. Create client.rb

So, it comes the time to create configuration file for `chef-client`, which is `client.rb` as mentioned above.

Specify the cookbook_path to search for cookbooks, which is `/tmp/chef/berks-cookbooks` in this case

```bash
#=> cat > /tmp/chef/client.rb <<'EOP'
cookbook_path File.join(Dir.pwd, 'berks-cookbooks')
EOP
```

Now, it works, can find and resolve dependency of cookbooks !

```bash
#=> chef-client -z -o 'mycookbook' -c /tmp/chef/client.rb 
Starting Chef Client, version 13.6.4
[2019-03-05T01:56:07+00:00] WARN: Run List override has been provided.
[2019-03-05T01:56:07+00:00] WARN: Run List override has been provided.
[2019-03-05T01:56:07+00:00] WARN: Original Run List: []
[2019-03-05T01:56:07+00:00] WARN: Original Run List: []
[2019-03-05T01:56:07+00:00] WARN: Overridden Run List: [recipe[mycookbook]]
[2019-03-05T01:56:07+00:00] WARN: Overridden Run List: [recipe[mycookbook]]
resolving cookbooks for run list: ["mycookbook"]
Synchronizing Cookbooks:
	......
  - mycookbook (0.0.0)
  ......
  - windows (3.4.1)
  - yum (5.0.1)
Installing Cookbook Gems:
Compiling Cookbooks...
```

***Note: if we use `berks vendor cookbooks`, it will download to `cookbooks` directory which is default for `chef-client` to look for local cookbooks, which mean the last step won't be neccessary if we use  `berks vendor cookbooks`***

[berkshelf]:https://docs.chef.io/berkshelf.html
[supermarket]:https://supermarket.chef.io