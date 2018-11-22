---
layout: post
title:  "Debugging LDAP for Gitlab"
date:   2018-10-24 10:27:50 +0800
categories: Gitlab
---

If LDAP login is not working for you, try following steps to debug.

# Start gitlab console
```shell
sudo gitlab-rails console
```

# Create a new gitlab ldap adapter
```ruby
adapter = Gitlab::Auth::LDAP::Adapter.new('ldapmain')
```

# Create ldap object
```ruby
ldap = Net::LDAP.new(adapter.config.adapter_options)
```

# Setup ldap search filter
```ruby
filter = Net::LDAP::Filter.eq("uid", "12345678")
```

# Search
```ruby
ldap.search(:base=>"ou=ldap,o=example.com",:filter=>filter)
```

# Print output message
```ruby
adapter.ldap.get_operation_result
```
If the search is good, you probally will see:
```console
=> #<OpenStruct code=0, message="Success">
```
Otherwise, you may see error like this:
```console
=> #<OpenStruct extended_response=nil, code=50, error_message="", matched_dn="", message="Insufficient Access Rights">
```
Then, probally, you need to look into further for the error.

The built-in rake command of gitlab could be a good one to start with, but it won't provide too much details.
```shell
sudo gitlab-rake gitlab:ldap:check

Checking LDAP ...

Server: ldapmain
LDAP authentication... Success
LDAP users with access to your GitLab server (only showing the first 100 results)

Checking LDAP ... Finished
```

Adding `--trace` could give you more detailed info
```shell
#=> gitlab-rake gitlab:ldap:check --trace
** Invoke gitlab:ldap:check (first_time)
** Invoke gitlab_environment (first_time)
** Invoke environment (first_time)
** Execute environment
** Execute gitlab_environment
** Execute gitlab:ldap:check
Checking LDAP ...
```
[Webhooks]: https://docs.gitlab.com/ee/security/webhooks.html
[apis]: https://docs.gitlab.com/ee/api/settings.html
