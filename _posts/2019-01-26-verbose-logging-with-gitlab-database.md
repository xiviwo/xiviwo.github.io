---
layout: post
title:  "Verbose Logging with Gitlab Database"
date:   2019-01-26 12:11:50 +0800
categories: Gitlab
---

Hereby I wish I could demostrate how to enable verbose logging for gitlab database issue.

### Cause

I try to upgrade our gitlab instance to 11.6.3, and face a error like: 

```console
ActiveRecord::StatementInvalid: PG::InsufficientPrivilege: ERROR:  permission denied to create extension "pg_trgm"
              HINT:  Must be superuser to create this extension.
              : CREATE EXTENSION IF NOT EXISTS "pg_trgm"

```

### Enable Debug Logging for Postgresql

The first thing that comes to my mind is to look into postgresql log. 

Generally for Red Hat based system, it's located at `var/lib/pgsql/data/postgresql.conf`

For Debian based system, it's located at `/etc/postgresql/<version>/main/postgresql.conf`

Add lines below to `postgresql.conf`
```conf
log_error_verbosity = 'VERBOSE'
log_line_prefix = '%t %u %d %a %p'
log_min_messages = debug5
log_min_error_statement = debug5
log_min_duration_statement = -1                                     
```

Restart postgres service for the change to take effect
```bash
systemctl restart postgresql-10
```

Probably, now you can see more verbose log like this:
```console
2019-01-25 07:26:18 GMT gitlab gitlabhq_production /opt/gitlab/embedded/bin/rake 61692ERROR:  42501: permission denied to create extension "pg_trgm"
2019-01-25 07:26:18 GMT gitlab gitlabhq_production /opt/gitlab/embedded/bin/rake 61692HINT:  Must be superuser to create this extension.
2019-01-25 07:26:18 GMT gitlab gitlabhq_production /opt/gitlab/embedded/bin/rake 61692LOCATION:  execute_extension_script, extension.c:809
2019-01-25 07:26:18 GMT gitlab gitlabhq_production /opt/gitlab/embedded/bin/rake 61692STATEMENT:  CREATE EXTENSION IF NOT EXISTS "pg_trgm"
```

From the log above, we can tell, extension creation needs superuser priviledge, but gitlab use is not superuser for our settings.

### Enable Verbose logging for Gitlab ActiveRecord

As well known, gitlab is based on famous `Ruby on Rails` framework, so if we need to look into database issue, we definitely need to focus on `ActiveRecord` of `Rails`.

So, `production` Rails configuration is located at `/opt/gitlab/embedded/service/gitlab-rails/config/environments/production.rb`.

#### Do it mannually

Basically, we just have to update/add two lines to the `production.rb`

```ruby
config.log_level = :debug
config.logger = Logger.new(STDOUT)
```


#### Do it in chef way

```ruby
ruby_block 'Enable ActiveRecord Debug Logging' do
  block do
    config_file = '/opt/gitlab/embedded/service/gitlab-rails/config/environments/production.rb'
    fe = Chef::Util::FileEdit.new(config_file)
    fe.search_file_replace_line(Regexp.escape('config.log_level = :info'), 'config.log_level = :debug')
    fe.write_file
  end
end

ruby_block 'Enable ActiveRecord Debug Logging to output to stdout' do
  block do
    config_file = '/opt/gitlab/embedded/service/gitlab-rails/config/environments/production.rb'
    fe = Chef::Util::FileEdit.new(config_file)
    fe.search_file_replace_line(Regexp.escape('# config.logger = ActiveSupport::TaggedLogging.new(SyslogLogger.new)'),'config.logger = Logger.new(STDOUT)')
    fe.write_file
  end
end
```

#### Do it in gitlab-rails console

Start gitlab rails console with command
```shell
sudo gitlab-rails console
```

Now, input native ruby command by:
```ruby
ActiveRecord::Base.logger = Logger.new(STDOUT)
```

#### Sample Debug Output of Gitlab ActiveRecord

```console
D, [2019-01-25T08:55:31.346877 #25679] DEBUG -- :    (169.7ms)  DROP DATABASE IF EXISTS "gitlabhq_production"
Dropped database 'gitlabhq_production'
D, [2019-01-25T08:55:31.671169 #25679] DEBUG -- :    (313.8ms)  CREATE DATABASE "gitlabhq_production" ENCODING = 'unicode' LC_COLLATE = ''
Created database 'gitlabhq_production'
-- enable_extension("plpgsql")
D, [2019-01-25T08:55:31.727890 #25679] DEBUG -- :   SQL (0.3ms)  CREATE EXTENSION IF NOT EXISTS "plpgsql"
   -> 0.0167s
-- enable_extension("pg_trgm")
D, [2019-01-25T08:55:31.756116 #25679] DEBUG -- :   SQL (25.8ms)  CREATE EXTENSION IF NOT EXISTS "pg_trgm"
   -> 0.0285s
```

### Conclusion

So, the answer to the issue we met in the first place is to create `pg_trgm` extension also for `template1` database of postgresql, so that all newly created table after that will be equiped with `pg_trgm` extension, without having to fight with permission issues. 


##### Sample chef code for create pg_trgm extension
```ruby
postgresql_extension 'create pg_trgm extension for template1' do
  database 'template1'
  extension 'pg_trgm'
  version '1.3'
end
```