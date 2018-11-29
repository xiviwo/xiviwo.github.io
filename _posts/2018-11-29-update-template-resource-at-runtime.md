---
layout: post
title:  "Update template variable at runtime"
date:   2018-11-29 17:02:50 +0800
categories: chef
---

Say we have a template resource as

```ruby
template node[:gitlab][:config] do
  variables(gitlab_rb: node[:gitlab][:gitlab_rb].to_hash )
  helper(:single_quote) { |value| value.nil? ? nil : "'#{value}'" }
  notifies :run, 'execute[gitlab-ctl reconfigure]', :immediately
  notifies :run, 'execute[gitlab-ctl restart]', :immediately
end
```
We hope to dynamically update `node[:gitlab][:gitlab_rb]` during runtime, and let configuration file `node[:gitlab][:config] or gitlab.rb` update at runtime as well

### Let's have a try with traditional way

We hope to add `registry_nginx['ssl_certificate']` to `gitlab.rb` when detecting discrepancy between certificate name and site name 

With this, unfortunately, it doesn't work.
```ruby
ruby_block 'if crt_name is not same as site name, configure certificate path' do
  action :run
  block do
    node.override[:gitlab][:gitlab_rb][:registry_nginx][:ssl_certificate] = crt_path
    node.override[:gitlab][:gitlab_rb][:registry_nginx][:ssl_certificate_key] = key_path
  end
  notifies :create, "template[#{node[:gitlab][:config]}]", :immediately
  notifies :run, 'execute[gitlab-ctl reconfigure]'
  only_if { crt_name != "#{site}.crt" }
end
```

As `variables(gitlab_rb: node[:gitlab][:gitlab_rb].to_hash )` has been determined during compilation time, it won't detect change during runtime.

As far as I am aware of there are at least two ways of doing that

### With Lazy variables evaluation



```ruby
template node[:gitlab][:config] do
  ei_perms :appsecure, :file
  # Evaluate variables during runtime
  variables(gitlab_rb: lazy { node[:gitlab][:gitlab_rb].to_hash })
  helper(:single_quote) { |value| value.nil? ? nil : "'#{value}'" }
  notifies :run, 'execute[gitlab-ctl reconfigure]', :immediately
  notifies :run, 'execute[gitlab-ctl restart]', :immediately
end
```

### With Chef Resource Update at runtime

Look at examples [here][gist] and [sethvargo][chef-resources]

```ruby
ruby_block 'if crt_name is not same as site name, configure certificate path' do
  action :run
  block do
    node.override[:gitlab][:gitlab_rb][:registry_nginx][:ssl_certificate] = crt_path
    node.override[:gitlab][:gitlab_rb][:registry_nginx][:ssl_certificate_key] = key_path
    r = resources(template: node[:gitlab][:config])
    r.variables(gitlab_rb: node[:gitlab][:gitlab_rb].to_hash )
  end
  notifies :run, 'execute[gitlab-ctl reconfigure]'
  only_if { crt_name != "#{site}.crt" }
end
```

[gist]: https://gist.github.com/arangamani/4659646
[chef-resources]: https://www.sethvargo.com/changing-chef-resources-at-runtime/