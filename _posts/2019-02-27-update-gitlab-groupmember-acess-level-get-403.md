---
layout: post
title:  "Update gitlab group member access_level get 403 Forbidden"
date:   2019-02-27 21:11:50 +0800
categories: Gitlab
---

### HTTP 403 Forbidden

Today, I try to update the 9th LDAP user of group 8, with `access_level=50`, that is [Owner access][OWNER] , but get `403 Forbidden` error. 

The API is showed as below. 

```console
RestClient.put "http://localhost/api/v4/groups/8/members/9?private_token=fJOED9DKSKdkeksjjd", "access_level=50", "Accept"=>"application/json", "Accept-Encoding"=>"gzip, deflate", "Content-Length"=>"15", "Content-Type"=>"application/x-www-form-urlencoded", "User-Agent"=>"rest-client/2.0.2 (linux-gnu x86_64) ruby/2.4.2p198"
# => 403 Forbidden | application/json 27 bytes
```

### Restriction for LDAP user permissions

So, we figure out that it's a new feature : [Updating user permissions - new feature][ldap]. 

It's said that LDAP user permissions must be overriden mannually by admin user, so the API call previously can run.

### One way
Also, we figure the operation can be done with general http call as below, but the drawback is that it's hard to find the corresponding relation between `id` below and the actual id of the member.

For example, the member id of previous API call is `9`, but the `id` below is 189 , which should be the id generated dynamically by gitlab along with webpage render.

Besides, you have to also deal with authentication for the http call below, which is headache.

```console
curl -X PATCH \
  http://localhost/groups/<group_name>/-/group_members/<id>/override \
  -H 'Content-Type: application/json' \
  -H 'cache-control: no-cache' \
  -d '{"group_member":{"override":true}}'
```

### Better way

So, if it has to be done with command or script, it can be done with embedded ruby script of gitlab

```ruby
#=> gitlab-rails runner -e production " \
	member = Group.find_by(id: group_id ).members.find_by(user_id: user_id) \
	member.override = true \
	member.save! \
"
```
After that, you should be seeing the previous API call working:

```console
RestClient.put "http://localhost/api/v4/groups/8/members/9?private_token=fJOED9DKSKdkeksjjd", "access_level=50", "Accept"=>"application/json", "Accept-Encoding"=>"gzip, deflate", "Content-Length"=>"15", "Content-Type"=>"application/x-www-form-urlencoded", "User-Agent"=>"rest-client/2.0.2 (linux-gnu x86_64) ruby/2.4.2p198"
# => 200 OK | application/json 820 bytes
```

[OWNER]: https://docs.gitlab.com/ee/api/access_requests.html
[ldap]: https://docs.gitlab.com/ee/administration/auth/how_to_configure_ldap_gitlab_ee/index.html#updating-user-permissions---new-feature