---
layout: post
title:  "Gitlab LFS Objects Storage Strategy"
date:   2019-04-11 09:38:50 +0800
categories: Gitlab
---

As we know, git is not designed to deal with large file storage(LFS), that is why [git lfs][lfs] comes in. 

But, it's not all the story, when we cerebrate the fully support to LFS of gitlab, another question pops up: Where and how are we going to store those LFS objects ? 

Basically, we believe there are two obvious options available, not exhaustive of course.

### Cloud Storage Sync Tool like Rclone

Rclone is a command line program to sync files and directories to and from various cloud object store provider, like famous as AWS S3, Google Cloud Storage, IBM COS S3, etc [rclone][rclone]

The fundamental idea is to keep multiple copys of LFS objects store, the first one is on your master gitlab instance, and master node will sync LFS objects to remote object store as backup periodically, and then over standby/slave gitlab instance the LFS objects will be pull down periodically as well

### Gitlab Native Solution

We are more interested in gitlab native solution, so called [Storing LFS objects in remote object storage][gitlab lfs administration], shall take length into a demo of that.


#### Add First LFS Object in Test Project

For example, over master gitlab instance

```sh
# Install git and git-lfs binary
yum install git -y 
yum install git-lfs -y

# clone the test project
git clone http://mygitlab.example.com/lfsproject/lfs-test.git
cd lfs-test

# initialize the Git LFS project
git lfs install

# Download whatever rpm as example
wget https://xxx.exmaple.com -O git-lfs-2.7.0-1.el7.x86_64.rpm

# select the file extensions that you want to treat as large files
git lfs track "*.rpm"

git add .
git commit -am 'add git-lfs-2.7.0-1.el7.x86_64.rpm'
git push origin master 
```

If we check against the new added file, we can tell clearly that one file is add and state is good
```console
[root@master lfs-test]# gitlab-rake gitlab:lfs:check VERBOSE=1
Checking integrity of LFS objects
- 1..1: Failures: 0
Done!
```


#### Enable LFS Objects Remote Store

over master gitlab instance

Edit `/etc/gitlab/gitlab.rb` , take `AWS s3` as example: 

***/etc/gitlab/gitlab.rb***
```ruby
gitlab_rails[:'lfs_object_store_background_upload'] = true
gitlab_rails[:'lfs_object_store_enabled'] = true
gitlab_rails[:'lfs_object_store_remote_directory'] = "<bucket name>/<folder/key name>"
gitlab_rails[:'lfs_object_store_connection'] = {
 'provider' => 'AWS',
 'region' => '<region>',
 'aws_access_key_id' => '<aws_access_key_id>',
 'aws_secret_access_key' => '<aws_secret_access_key>',
 # The below options configure an S3 compatible host instead of AWS
 'host' => 'localhost',
 'endpoint' => '<endpoint of s3 >',
 'path_style' => true
}
```
Reconfigure and restart gitlab for the changes to take effect
 

#### Add Another File to the Test Project
 
```sh
wget https://d28dx6y1hfq3.example.com -O git-lfs-2.6.1-1.el7.x86_64.rpm
git add .
git commit -am 'add git-lfs-2.6.1-1.el7.x86_64.rpm'
git push origin master 

```

if we check the lfs object state again, both object are good
 
```sh
[root@master lfs-test]# gitlab-rake gitlab:lfs:check VERBOSE=1
Checking integrity of LFS objects
- 1..2: Failures: 0
Done!
```
 
But, if we check over standby node, both failed.
 
It makes sense, because the first object is not transferred to standby node, and the second object, which should be on remote object store, should not be available to standby node now, as the lfs objects remote store is not enabled for standby node yet.
 
```sh
[root@standby ~]# gitlab-rake gitlab:lfs:check VERBOSE=1
Checking integrity of LFS objects
- 1..2: Failures: 2
  - LFS object: e2604780fa145af0f0458fa145af0f0453f187cf8dca40f3fac327: #<Errno::ENOENT: No such file or directory @ rb_sysopen - /var/opt/gitlab/lfs-objects/e2/60/e2604780fa145af0f0458fa145af0f0453f187cf8dca40f3fac327>
  - LFS object: 6fe33c5cba2aa0585c2626cfc8e966b1fc07724ec0ea0585c2626c: #<RuntimeError: Object Storage is not enabled>
Done!
```
 

#### Enable the LFS Objects Remote Store on Standby
 
We got one failure, which also make sense, as only the second object is available on remote store, it can now reach the second object, so only one failure is identifed.
 
```sh
[root@standby ~]# gitlab-rake gitlab:lfs:check VERBOSE=1
Checking integrity of LFS objects
- 1..2: Failures: 1
  - LFS object: e2604780fa145af0f0458fa145af0f0453f187cf8dca40f3fac327: #<Errno::ENOENT: No such file or directory @ rb_sysopen - /var/opt/gitlab/lfs-objects/e2/60/4780fa145af0f0458c221adbb8316c93f187cf8dca40f3fac327e41ca0dd>
Done!
```


#### Migrate any Existing local LFS objects to the Object Storage

If  we migrate any existing local LFS objects to the object storage, those are existing before lfs objects remote store is enabled, command can be ran as :
 
```sh
[root@master lfs-test]# gitlab-rake gitlab:lfs:migrate
I, [2019-04-10T12:07:43.050228 #25268]  INFO -- : Starting transfer of LFS files to object storage
I, [2019-04-10T12:07:44.547204 #25268]  INFO -- : Transferred LFS object e2604780fa145af0f0458fa145af0f0453f187cf8dca40f3fac327 of size 3198212 to object storage
```

Now, check again over standby node, both objects are good now.

```sh
[root@standby ~]# gitlab-rake gitlab:lfs:check VERBOSE=1
Checking integrity of LFS objects
- 1..2: Failures: 0
Done!
```

#### What if We Need to Sync Back? 

So, gitlab only provides `gitlab-rake gitlab:lfs:migrate` command to sync local to remote as way of manual migration.

What if we need to sync remote to local ? 

Luckly, it can be done with `gitlab-rails console`

run `gitlab-rails console` in the bash shell

```sh
sudo gitlab-rails console
```

Paste lines below, it will works

```ruby
logger = Logger.new(STDOUT)

LfsObject.with_files_stored_remotely.find_each(batch_size: 10) do |lfs_object|
  begin
    lfs_object.file.migrate!(LfsObjectUploader::Store::LOCAL)

    logger.info("Transferred LFS object #{lfs_object.oid} of size #{lfs_object.size.to_i.bytes} to local")
  rescue => e
    logger.error("Failed to transfer LFS object #{lfs_object.oid} with error: #{e.message}")
  end
end
```

### Conclusion 

So, you can have more control over option #1, and flexibility as you like if you go option #1;


For option #2, it's native feature of gitlab, it's nice, simple and easy to configure and use, you don't have to worry a lot after setup.

But definitely you can't control must of the logic underneath, and could be a problem if you have special use case and requirement. 


[lfs]:http://git-lfs.github.com
[rclone]:https://rclone.org/
[gitlab lfs administration]:https://docs.gitlab.com/ee/workflow/lfs/lfs_administration.html#storing-lfs-objects-in-remote-object-storage