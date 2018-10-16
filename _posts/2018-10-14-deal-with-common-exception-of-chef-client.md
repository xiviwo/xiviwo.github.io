---
layout: post
title:  "Deal With Common Exception of Chef-client"
date:   2018-10-14 08:41:50 +0800
categories: Chef
---
Well, for our chef client, we are seeing continuously error like below, that is killing out chef run.
```bash
error: rpmdb: BDB0113 Thread/process 4149/140556735592512 failed: BDB1507 Thread died Berkeley DB library
error: db5 error(-30973) from dbenv->failchk: BDB0087 DB_RUNRECOVERY: Fatal error, run database recovery
error: cannot open Packages index using db5 -  (-30973)
error: cannot open Packages database in /var/lib/rpm
error: rpmdb: BDB0113 Thread/process 4149/140556735592512 failed: BDB1507 Thread died in Berkeley DB library
error: db5 error(-30973) from dbenv->failchk: BDB0087 DB_RUNRECOVERY: Fatal error, run database recovery
error: cannot open Packages database in /var/lib/rpm
```

So, basically, we are seeing a lot of this, we can fix by:

```bash
rm -rvf /var/lib/rpm/__db*
rpm --rebuilddb
```

But,I am tired of doing this again and again, why come up with something automatic ? 

So, here it comes a simple demo:

Firstly, we create a simple module in the `libraries` folder:


***rpm_db_handler.rb***
```ruby
module RPMDBHandler
  # helper for rpmdb fix
  class Helper
    def fix_rpmdb_on_db_corrupt(message)
      return false unless message.include?('cannot open Packages database in /var/lib/rpm')
      puts 'Found Corrupted RPMDB, try to fix.'
      puts `rm -rvf /var/lib/rpm/__db*`
      puts 'rpm --rebuilddb'
      puts `rpm --rebuilddb`
    end
  end
end
```

Then, create a demo recipe, and add code like this:

***default.rb***
```ruby
Chef.event_handler do
  on :run_failed do |exception|
    RPMDBHandler::Helper.new.fix_rpmdb_on_db_corrupt(exception.message)
  end
end

ruby_block 'fail the run' do
  block do
    fail 'cannot open Packages database in /var/lib/rpm'
  end
end
```

Then, if you run `kitchen converge`, you should be seeing it's working as expected.
It means if same error happens again, it can be fixed automatically.

```
Running handlers:
[2018-10-13T15:00:31+00:00] ERROR: Running exception handlers
[2018-10-13T15:00:31+00:00] ERROR: Running exception handlers
Running handlers complete
[2018-10-13T15:00:31+00:00] ERROR: Exception handlers complete
[2018-10-13T15:00:31+00:00] ERROR: Exception handlers complete
Chef Client failed. 0 resources updated in 02 seconds
Found Corrupted RPMDB, try to fix.
removed '/var/lib/rpm/__db.001'
removed '/var/lib/rpm/__db.002'
removed '/var/lib/rpm/__db.003'
rpm --rebuilddb

[2018-10-13T15:00:33+00:00] FATAL: Stacktrace dumped to /tmp/kitchen/cache/chef-stacktrace.out
[2018-10-13T15:00:33+00:00] FATAL: Stacktrace dumped to /tmp/kitchen/cache/chef-stacktrace.out
```

So, that is it. But, still need to figure out how to apply globally....