---
layout: post
title:  "About postgresql_access"
date:   2018-10-18 10:02:50 +0800
categories: Gitlab
---

So, again, let's review how to use postgresql_access resource of [postgres 7.1.1 cookbook][postgresql cookbook].

```ruby
[{ type: 'local', db: 'all', user: 'postgres', addr: nil, method: 'ident' },
 { type: 'local', db: 'replication', user: 'postgres', addr: nil, method: 'ident' },
 { type: 'local', db: 'all', user: 'all', addr: nil, method: 'peer map=gitlab' }, 
 { type: 'host', db: 'replication', user: 'replication', addr: '0.0.0.0/0', method: 'md5' }].each do |item|
  postgresql_access "set up postgresql access for #{item[:user]}" do
    access_type item[:type]
    access_db item[:db]
    access_user item[:user]
    access_addr item[:addr]
    access_method item[:method]
  end
end
```
Previously, code block above yield configure file like below:

***pg_hba.conf***
```conf
# This file was automatically generated and dropped off by Chef!

# PostgreSQL Client Authentication Configuration File
# ===================================================
#
# Refer to the "Client Authentication" section in the PostgreSQL
# documentation for a complete description of this file.

local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5

###########
# From the postgresql_access resources
###########
# local-all-postgres-ident
local   all             postgres                                ident
# local-replication-postgres-ident
local   replication     postgres                                ident
# local-all-all-peer map=gitlabmapping
local   all             all                                     peer map=gitlabmapping
# host-replication-replication-md5
host    replication     replication     0.0.0.0/0               md5
```
Yes, that likes good and nice, but it gave me error after postgres restart.
```log
DETAIL:  Connection matched pg_hba.conf line 14: "local   all   all    peer"
LOG:  provided user name (gitlab) and authenticated user name (git) do not match
```

let's take a look at what `pg_ident.conf` looks like:
```conf

# MAPNAME       SYSTEM-USERNAME         PG-USERNAME

# webinst Mapping
gitlabmapping          git                     gitlab         

# gitlab-ci Mapping
gitlabmapping          gitlab-ci               gitlab-ci      

# /^(.*)$ Mapping
gitlabmapping          /^(.*)$                 \1     
```

So, we are system user `git`, and hoping to act as pg-user `gitlab`, according to `pg_ident.conf`.

And, rule `local   all  all    peer map=gitlabmapping` in `pg_hba.conf`, trys to map `git` system-user to `gitlab` pg-user with the mapname of `gitlabmapping` in `pg_ident.conf`, in order to communicate with postgres.

So, we are expecting rule `local   all  all    peer map=gitlabmapping`, but we got rule `local   all   all    peer`, why ?

The key is the order, rule `local   all   all    peer` appears firstly in `pg_hba.conf`, so it get matched firstly.

So, the solution is to change the order of the rule, how ?

Well, as far as I can tell, you have to provide template of your own, for resource postgresql_access.

So, create a template for your cookbook.

***templates/_pg_hba.conf.erb***
```conf
# This file was automatically generated and dropped off by Chef!

# PostgreSQL Client Authentication Configuration File
# ===================================================
#
# Refer to the "Client Authentication" section in the PostgreSQL
# documentation for a complete description of this file.

# TYPE  DATABASE        USER            ADDRESS                 METHOD


###########
# Other authentication configurations taken from chef node defaults:
###########
<% node['postgresql']['pg_hba'].each do |auth| -%>

<%   if auth[:comment] %>
# <%= auth[:comment] %>
<%   end %>
<%   if auth[:addr] %>
<%= auth[:type].ljust(7) %> <%= auth[:db].ljust(15) %> <%= auth[:user].ljust(15) %> <%= auth[:addr].ljust(23) %> <%= auth[:method] %>
<%   else %>
<%= auth[:type].ljust(7) %> <%= auth[:db].ljust(15) %> <%= auth[:user].ljust(15) %>                         <%= auth[:method] %>
<%   end %>
<% end %>

# "local" is for Unix domain socket connections only
local   all             all                                     peer

```
Update postgresql_access code block as below:
```ruby
  postgresql_access "#{item[:type]}-#{item[:db]}-#{item[:user]}-#{item[:method]}" do
    # put '_pg_hba.conf.erb' in your 'templates/_pg_hba.conf.erb'
    source '_pg_hba.conf.erb'
    # cookbook directive is necessary for postgresql_access to look up where is your templates
    cookbook 'your cookbook name'
    access_type item[:type]
    access_db item[:db]
    access_user item[:user]
    access_addr item[:addr]
    access_method item[:method]
  end
```
It should yield new `pg_hba.conf` for you.
```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD


###########
# Other authentication configurations taken from chef node defaults:
###########

local   all             postgres                                ident

local   replication     postgres                                ident

local   all             all                                     peer map=gitlabmapping

host    replication     replication     0.0.0.0/0               md5

# "local" is for Unix domain socket connections only
local   all             all                                     peer
```
Restart postgresql to take effect
```bash
$ systemctl restart postgresql-10.service
```
By now, postgresql should be happy and working now.

[postgresql cookbook]: https://supermarket.chef.io/cookbooks/postgresql/versions/7.1.1
