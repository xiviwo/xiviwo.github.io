---
layout: post
title:  "How to restore gitlab db ONLY"
date:   2019-04-13 19:17:50 +0800
categories: Gitlab
---

Well, you may run into scenarios like that sooner or later: it's well known how to gitlab as a [whole][restore], but how to restore gitlab db ONLY ?

It looks like there is no official document or third-party note anywhere, so I look in to the source code of gitlab.

### Trial and Error I

So, the version of gitlab-ee I am looking in is `11.6.0-pre`, and the `backup.rake` file is located at `gitlab-ee/lib/tasks/gitlab/backup.rake`

For the `gitlab:backup:create` namespace, we can easily identified a lot of sub tasks consisted of gitlab backup command, so since there is `gitlab:backup:db:create`, we can conclude that must be `gitlab:backup:db:restore` available somewhere

```ruby
Rake::Task["gitlab:backup:db:create"].invoke
Rake::Task["gitlab:backup:repo:create"].invoke
Rake::Task["gitlab:backup:uploads:create"].invoke
Rake::Task["gitlab:backup:builds:create"].invoke
Rake::Task["gitlab:backup:artifacts:create"].invoke
Rake::Task["gitlab:backup:pages:create"].invoke
Rake::Task["gitlab:backup:lfs:create"].invoke
Rake::Task["gitlab:backup:registry:create"].invoke
```
If we search with `gitlab:backup:db:restore`, it instantly get me to these codes

```ruby
  # Drop all tables Load the schema to ensure we don't have any newer tables
  # hanging out from a failed upgrade
  progress.puts 'Cleaning the database ... '.color(:blue)
  Rake::Task['gitlab:db:drop_tables'].invoke
  progress.puts 'done'.color(:green)
  Rake::Task['gitlab:backup:db:restore'].invoke
rescue Gitlab::TaskAbortedByUserError
  puts "Quitting...".color(:red)
  exit 1
end
```

I can easily tell from the code, it tried to drop table before restoring with `gitlab:db:drop_tables`.
So,it leds me to some instructions like these combining official documents:

```sh
sudo cp 1555128457_2019_04_13_11.6.3-ee_gitlab_backup.tar /var/opt/gitlab/backups/
sudo chown git.git /var/opt/gitlab/backups/1555128457_2019_04_13_11.6.3-ee_gitlab_backup.tar

sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
# Verify
sudo gitlab-ctl status

gitlab-rake gitlab:db:drop_tables
gitlab-rake gitlab:backup:db:restore BACKUP=1555128457_2019_04_13_11.6.3-ee force=yes

sudo gitlab-ctl restart
sudo gitlab-rake gitlab:check SANITIZE=true

```

run and test, but it gave me :

```sh
Restoring database ... 
rake aborted!
Errno::ENOENT: No such file or directory - /var/opt/gitlab/backups/db/database.sql.gz
/opt/gitlab/embedded/service/gitlab-rails/lib/backup/database.rb:54:in `spawn'
/opt/gitlab/embedded/service/gitlab-rails/lib/backup/database.rb:54:in `restore'
```
### Trial and Error II

Clearly, I need to untar the backup file to see what will happen

Untar by :

```sh
tar xf 1555128457_2019_04_13_11.6.3-ee_gitlab_backup.tar
```

check file, look like we have `/var/opt/gitlab/backups/db/database.sql.gz` now. 
```sh
[root@gitlab backups]# ls
1555128457_2019_04_13_11.6.3-ee_gitlab_backup.tar  backup_information.yml  db            repositories
artifacts.tar.gz                                   builds.tar.gz           pages.tar.gz  uploads.tar.gz
[root@gitlab backups]# ls db/
database.sql.gz
```

Rerun the instruction again, everything works like a charm !


[restore]:https://docs.gitlab.com/ee/raketasks/backup_restore.html#restore-for-omnibus-gitlab-installations