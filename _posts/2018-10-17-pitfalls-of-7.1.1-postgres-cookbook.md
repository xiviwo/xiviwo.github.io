---
layout: post
title:  "Pitfalls of the 7.1.1 Postgresql Cookbook"
date:   2018-10-17 10:52:50 +0800
categories: Chef
---

Also, we decide to upgrade postgres installation as well, so I found the latest postgresql cookbook `7.1.1` [here][postgresql cookbook].

It looks nice and clean at first glance, as it converts all recipes into resources:

- postgresql_client_install
- postgresql_server_install
- postgresql_server_conf
- postgresql_extension
- postgresql_access
- postgresql_ident
- postgresql_database
- postgresql_user

It claims you can install postgres server with simpler code, like:
```ruby
postgresql_server_install 'My Postgresql Server install' do
  action :install
end

postgresql_server_install 'Setup my postgresql 9.5 server' do
  version '9.5'
  password 'MyP4ssw0d'
  port 5433
  action :create
end
```
But, when it comes to real field, you will see a lot of trouble.

# No easy way to override data dir
Previously, data dir is easy to override, just with one line code, you are done.
```ruby
override[:postgresql][:dir] = '/var/opt/gitlab/pgsql/data'
```
But, now the data dir or conf dir is nearly hardcoded there, you can't override with general chef approach.
```ruby
def data_dir(version = node.run_state['postgresql']['version'])
  case node['platform_family']
  when 'rhel', 'fedora'
    "/var/lib/pgsql/#{version}/data"
  when 'amazon'
    if node['virtualization']['system'] == 'docker'
      "/var/lib/pgsql#{version.delete('.')}/data"
    else
      "/var/lib/pgsql/#{version}/data"
    end
  when 'debian'
    "/var/lib/postgresql/#{version}/main"
  end
end

def conf_dir(version = node.run_state['postgresql']['version'])
  case node['platform_family']
  when 'rhel', 'fedora'
    "/var/lib/pgsql/#{version}/data"
  when 'amazon'
    if node['virtualization']['system'] == 'docker'
      "/var/lib/pgsql#{version.delete('.')}/data"
    else
      "/var/lib/pgsql/#{version}/data"
    end
  when 'debian'
    "/etc/postgresql/#{version}/main"
  end
end
```
Well, I have to override elsewise, create a module in the `libraries` folder, override the method in `PostgresqlCookbook::Helpers` in the upstream cookbook.

***postgres_helper.rb***
```ruby
module PostgresqlCookbook
  module Helpers
    def data_dir(_version = node[:postgresql][:version])
      node[:postgresql][:dir]
    end

    def conf_dir(_version = node[:postgresql][:version])
      node[:postgresql][:dir]
    end
  end
end

Chef::Recipe.send(:include, PostgresqlCookbook::Helpers)
```
Well, I know, it looks ugly, it should do the trick

# Not creating 'postgres' user by default, not creating data dir, do it yourself
Previously, it's all done within 
```ruby
include_recipe 'postgresql::server'
```

Now, you have to do
```ruby
postgres_user = node[:postgresql][:user]

group postgres_user do
  gid '26'
end

user postgres_user do
  shell '/bin/bash'
  comment 'PostgreSQL Server'
  home '/var/lib/pgsql'
  gid postgres_user
  system true
  uid '26'
  manage_home false
end

directory node[:postgresql][:dir] do
  owner postgres_user
  group postgres_user
  recursive true
  action :create
  mode '0700'
end
```

# Mysterious "undefined method [] for nil:NilClass"
It always throws NoMethodError for `postgresql_server_install`, something like:
```bash
NoMethodError: undefined method `[]' for nil:NilClass
/tmp/kitchen/cache/cookbooks/postgresql/libraries/helpers.rb:155:in `conf_dir'
```
It turn out to be `node.run_state['postgresql']` is nil.
```ruby
def conf_dir(version = node.run_state['postgresql']['version'])
```
Not sure what cause this, but there is one workaround
```ruby
ruby_block 'fix_for_postgresql_cookbook_node_run_state' do
  block do
    node.run_state['postgresql'] ||= {}
    node.run_state['postgresql']['version'] = node[:postgresql][:version]
  end
end

```

So, you are done with postgres cookbook? No, when you try to start service of postgres, it will tell you, `/var/lib/pgsql/10/data/` is missing or empty. 

Hold on, data dir should be overrided by our module trick, the folder is not correct, should point to `/var/opt/gitlab/pgsql/data`, so what is happening?
```bash
Starting PostgreSQL 10 database server...
postgresql-10-check-db-dir[4002]: "/var/lib/pgsql/10/data/" is missing or empty.
```
# Override Systemd Unit File
So, the answer is you have to create systemd unit of your own and set up correct data dir.

Luckily, you have example to follow, `/lib/systemd/system/postgresql-10.service`.

Basically, you just have to replace two line of it.
```ini

  # Location of database directory
  Environment=PGPORT=#{node[:postgresql][:port]}
  Environment=PGDATA=#{node[:postgresql][:dir]}

```

# Full Sources

***attributes/default.rb***
```ruby
default[:postgresql][:pg_hba] = [{ type: 'local', db: 'all', user: 'postgres', addr: nil, method: 'ident' },
                                 { type: 'local', db: 'replication', user: 'postgres', addr: nil, method: 'ident' },
                                 { type: 'local', db: 'all', user: 'all', addr: nil, method: 'peer map=gitlab' },
                                 { type: 'host', db: 'replication', user: 'replication', addr: '0.0.0.0/0', method: 'md5' }]
default[:postgresql][:pg_ident] = [{ mapname: 'gitlab', systemuser: 'glitab', pguser: 'gitlab' },
                                   { mapname: 'gitlab', systemuser: 'gitlab-ci', pguser: 'gitlab-ci' },
                                   { mapname: 'gitlab', systemuser: '/^(.*)$', pguser: '\1' }]

default[:postgresql][:version] = '10'
default[:postgresql][:dir] = '/var/opt/gitlab/pgsql/data'
default[:postgresql][:password][:postgres] = 'md32d8c9283920020298ecdsi923kdkf'
default[:postgresql][:user] = 'postgres'
default[:postgresql][:port] = 5432
```

***libraries/postgres_helper.rb***
```ruby
module PostgresqlCookbook
  module Helpers
    def data_dir(_version = node[:postgresql][:version])
      node[:postgresql][:dir]
    end

    def conf_dir(_version = node[:postgresql][:version])
      node[:postgresql][:dir]
    end
  end
end

Chef::Recipe.send(:include, PostgresqlCookbook::Helpers)
```

***recipes/default.rb***
```ruby
postgres_user = node[:postgresql][:user]

group postgres_user do
  gid '26'
end

user postgres_user do
  shell '/bin/bash'
  comment 'PostgreSQL Server'
  home '/var/lib/pgsql'
  gid postgres_user
  system true
  uid '26'
  manage_home false
end

directory node[:postgresql][:dir] do
  owner postgres_user
  group postgres_user
  recursive true
  action :create
  mode '0700'
end

ruby_block 'fix_for_postgresql_cookbook_node_run_state' do
  block do
    node.run_state['postgresql'] ||= {}
    node.run_state['postgresql']['version'] = node[:postgresql][:version]
  end
end

postgresql_server_install 'Install Postgres Server' do
  version node[:postgresql][:version]
  action :install
end

systemd_unit "postgresql-#{node[:postgresql][:version]}.service" do
  content <<-EOU.gsub(/^\s+/, '')
  [Unit]
  Description=PostgreSQL 10 database server
  Documentation=https://www.postgresql.org/docs/10/static/
  After=syslog.target
  After=network.target

  [Service]
  Type=notify

  User=postgres
  Group=postgres

  # Note: avoid inserting whitespace in these Environment= lines, or you may
  # break postgresql-setup.

  # Location of database directory
  Environment=PGPORT=#{node[:postgresql][:port]}
  Environment=PGDATA=#{node[:postgresql][:dir]}

  # Where to send early-startup messages from the server (before the logging
  # options of postgresql.conf take effect)
  # This is normally controlled by the global default set by systemd
  # StandardOutput=syslog

  # Disable OOM kill on the postmaster
  OOMScoreAdjust=-1000
  Environment=PG_OOM_ADJUST_FILE=/proc/self/oom_score_adj
  Environment=PG_OOM_ADJUST_VALUE=0

  ExecStartPre=/usr/pgsql-10/bin/postgresql-10-check-db-dir ${PGDATA}
  ExecStart=/usr/pgsql-10/bin/postmaster -D ${PGDATA}
  ExecReload=/bin/kill -HUP $MAINPID
  KillMode=mixed
  KillSignal=SIGINT


  # Do not set any timeout value, so that systemd will not kill postmaster
  # during crash recovery.
  TimeoutSec=0

  [Install]
  WantedBy=multi-user.target

  EOU
  triggers_reload true
  action %i(create enable)
end

postgresql_server_install 'Init Postgres Server' do
  version node[:postgresql][:version]
  password node[:postgresql][:password][:postgres]
  ident_file "#{node[:postgresql][:dir]}/pg_ident.conf"
  hba_file "#{node[:postgresql][:dir]}/pg_hba.conf"
  initdb_locale 'en_US.UTF-8'
  port node[:postgresql][:port]
  action :create
end

node[:postgresql][:pg_ident].each do |item|
  postgresql_ident "Map #{item[:systemuser]} to #{item[:pguser]}" do
    comment "#{item[:systemuser]} Mapping"
    mapname item[:mapname]
    system_user item[:systemuser]
    pg_user item[:pguser]
  end
end

node[:postgresql][:pg_hba].each do |item|
  postgresql_access "#{item[:type]}-#{item[:db]}-#{item[:user]}-#{item[:method]}" do
    access_type item[:type]
    access_db item[:db]
    access_user item[:user]
    access_addr item[:addr]
    access_method item[:method]
  end
end

postgresql_server_conf 'Update Postgres Config' do
  version node[:postgresql][:version]
  data_directory node[:postgresql][:dir]
  ident_file "#{node[:postgresql][:dir]}/pg_ident.conf"
  hba_file "#{node[:postgresql][:dir]}/pg_hba.conf"
  notifies :restart, "service[postgresql-#{node[:postgresql][:version]}]", :immediately
end

service "postgresql-#{node[:postgresql][:version]}" do
  supports restart: true, status: true, reload: true
  action %i(enable start)
end

postgresql_user 'gitlab' do
  create_user 'gitlab'
  user postgres_user
  password 'P@ssw0rd'
  createdb true
  login true
  inherit false
  sensitive false
end

postgresql_database 'Create Gitlab Database' do
  database 'gitlab_production'
  user postgres_user
  owner 'gitlab'
end

postgresql_extension 'Postgres pg_trgm' do
  database 'gitlab_production'
  extension 'pg_trgm'
  version '1.3'
end
```


[postgresql cookbook]: https://supermarket.chef.io/cookbooks/postgresql/versions/7.1.1
