---
layout: post
title:  "Manipulate gitlab object with gitlab-rails console"
date:   2018-12-04 22:22:50 +0800
categories: gitlab
---

It's very interesting to manipulate gitlab object directly with gitlab console. 

As gitlab is based on `Ruby on Rails`, so if you once work with `Ruby on Rails`, the following steps can look quite familiar to you though.

First time firstly, bring up a console with

```console
#=> sudo gitlab-rails console
```

#### Create one admin user

```ruby
user = User.create(:username => 'test_admin', :name => 'Administrator', :password => 'very_secure_password', :password_confirmation => 'very_secure_password', :admin => true, :email => 'admin@example.com',:confirm => false); 
user.save!
```

#### Or, change the password of admin user

```ruby
root = User.find_by(username: "root")
root.password = 'update_password'
root.password_confirmation = 'update_password'
root.save!
```

#### Or, list the attributes or methods of user

```ruby
user = User.find_by(username: "root")
user.attributes
user.methods.sort
```

#### List all users
```ruby
User.all
=> #<ActiveRecord::Relation [#<User id:6 @test6>, #<User id:5 @test5>, #<User id:1 @root>, #<User id:2 @test3>, #<User id:4 @test4>]>
```

#### Or, even, delete user. DANGEROUS! DO IT WITH CAUTIONS!

```ruby
user = User.find_by(username: "test")
user.delete
```
#### Confirm users

```ruby
user = User.find_by('test')
user.confirmed_at = Time.now
user.confirmation_token = nil
user.save!
```

#### List license

```ruby
license = License.find_by(id:1)
license.data
```
