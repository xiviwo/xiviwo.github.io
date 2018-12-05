---
layout: post
title:  "Query Chef Server Embedded Postgres DB"
date:   2018-12-05 16:37:50 +0800
categories: [Chef,Postgresql]
---

#### Built-in way

Sometime we need to look into the embedded `PostgreSQL` db of Chef server, we have handy built-in command, for example

```console
#=> echo 'SELECT version()' |  chef-server-ctl psql opscode_chef
                                                 version                                                  
----------------------------------------------------------------------------------------------------------
 PostgreSQL 9.2.15 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.3 20140911 (Red Hat 4.8.3-9), 64-bit
(1 row)

```

It querys `opscode_chef` service for the current version of `PostgreSQL` we are working with.

 Valid names include: ["opscode_chef", "bifrost", "oc_id", "bookshelf", "push-jobs", "reporting", "oc_erchef", "oc-id", "opscode-erchef"] depending on what services are enabled.

Or run as interact session as:

```console
#=> chef-server-ctl psql opscode_chef
psql (9.2.15)
Type "help" for help.

opscode_chef=> \d
                               List of relations
 Schema |                    Name                    |   Type   |     Owner     
--------+--------------------------------------------+----------+---------------
 public | checksums                                  | table    | opscode-pgsql
 public | clients                                    | table    | opscode-pgsql
 public | containers                                 | table    | opscode-pgsql
 ......
```

#### Hack way

 But, if you want to have more control over what is under the hood, let's do it in native sql way.

 By default, the `psql` binary is located at `/opt/opscode/embedded/postgresql/9.2/bin/psql` and link to `/opt/opscode/embedded/bin/psql`, you will be better off with `/opt/opscode/embedded/bin/psql`, since different version of chef server comes with different version of `PostgreSQL`, `9.2` in our case

 But, where can we get the username and password ? 

 Yeah, it's located at `/etc/opscode/chef-server-running.json`, where all the chef running settings and configuration are stored.

 For example, run command below will yield the pq user name and password.

```console
 #=> python -c "import json; a = json.loads(open('/etc/opscode/chef-server-running.json').read());print 'sql_user=',a[u'private_chef']['opscode-erchef']['sql_user'];print 'sql_password=',a[u'private_chef']['opscode-erchef']['sql_password']"
sql_user= opscode_chef
sql_password= 9329239329cd86a34cc9d8sdj1da05d8sd8ds920sd9ds8918cb8b55dccb7
 ```

Now, you can achieve same result with:

```console
#=> PGPASSWORD=9329239329cd86a34cc9d8sdj1da05d8sd8ds920sd9ds8918cb8b55dccb7 /opt/opscode/embedded/bin/psql -p 5432 -U opscode_chef -d opscode_chef -h localhost -t -c 'SELECT version()'
 PostgreSQL 9.2.15 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.3 20140911 (Red Hat 4.8.3-9), 64-bit
```

Of course, it's not recommended way to specify password by `PostgreSQL` as [here][pgpass]

```console
#=> cat > ~/.pgpass <<'EOF'
localhost:5432:opscode_chef:opscode_chef:9329239329cd86a34cc9d8sdj1da05d8sd8ds920sd9ds8918cb8b55dccb7
EOF
#=> chmod 0600 /root/.pgpass
```
Now, you can query without password

```console
#=> /opt/opscode/embedded/bin/psql -U opscode_chef -h localhost -t -c 'SELECT version()'
PostgreSQL 9.2.15 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.3 20140911 (Red Hat 4.8.3-9), 64-bit
```

If you want to get rid of `opscode_chef` and `localhost` parameters completely, `pg_hba.conf` must be configured to map between `opscode_chef` and `localhost`, like 

```conf
# pg_hba.conf
# TYPE DATABASE USER CIDR-ADDRESS  METHOD
host  all  opscode_chef 127.0.0.1/32 md5
```

[pgpass]: https://www.postgresql.org/docs/9.0/libpq-pgpass.html