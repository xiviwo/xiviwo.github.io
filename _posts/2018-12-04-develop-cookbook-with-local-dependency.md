---
layout: post
title:  "Develop Chef Cookbook with local dependency"
date:   2018-12-04 10:54:50 +0800
categories: chef
---

### Issue with Berkshelf 

If you work with `Berkshelf`, you might meet such scenario: 

we update the upstream base cookbook, but would like to test with the downstream wrapper cookbook at the same time. But, the base cookbook is still under development, we don't want to upload to the chef server until it's verified. And, the wrapper cookbook is supposed to require some specific version of base cookbook from chef server. But, we can't tell if all these will work, unless we bundle the base cookbook with the wrapper cookbook and test.

### How to deal with that? 

By default, `berkshelf` looks for cookbook from server, with `source` keyword, like below

```ruby
source "https://supermarket.chef.io"
source :chef_server
```

We can utilize the `Cookbook Keyword` from `Berksfile`, [here][berkshelf], which takes syntax like, `cookbook "NAME" [, "VERSION_CONSTRAINT"] [, SOURCE_OPTIONS]` and deliberately `source` or `require` local base cookbook with:

```ruby
cookbook('base-cookbook', path: File.expand_path('../base-cookbook', __dir__)) 
```

if we run `berks` again, we will see it's sourcing from `../base-cookbook` now. looks good.

```console
#=> /opt/chefdk/bin/berks
Resolving cookbook dependencies...
Fetching 'base-cookbook' from source at ../base-cookbook
```

### Direct dependency vs. Indirect dependency

In fact, it will only work with direct dependency.

For example:

B depends on A

A depends on C

D depends on C

B depends on D

A/B wants to source local version of C

D want to source `~> 3.0` of C

It will cause `conflicting dependencies`, as `D` require `(C ~> 3.0)` but `A/B` require `(c = 0.0.0)`,(local cookbook, has no version)

In that case, we could do some trick to make `D` believe it's sourcing `C  ~> 3.0`.

Looking into the `metadata.rb` of C, we can see:

```ruby
version          IO.read(File.join(File.dirname(__FILE__), 'VERSION')) rescue '0.0.1'
```

we can trick it into something like below since it's only for test temporily
```ruby
version          '3.2.2' #IO.read(File.join(File.dirname(__FILE__), 'VERSION')) rescue '0.0.1'
```

Run `berks` again, we got:
```console
/opt/chefdk/bin/berks 
Resolving cookbook dependencies...
Fetching 'base-cookbook' from source at ../base-cookbook
.....
Fetching cookbook index from chef_server: https://chef.example.com/organizations/org...
.....
.....
Using base-cookbook (3.2.2) from source at ../base-cookbook
```

So, `D` recognizes `../base-cookbook` as `base-cookbook (3.2.2)`, it works !



[berkshelf]: https://docs.chef.io/berkshelf.html#cookbook-keyword