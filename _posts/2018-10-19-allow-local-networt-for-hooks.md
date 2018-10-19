---
layout: post
title:  "Allow requests to the local network from hooks"
date:   2018-10-19 11:06:50 +0800
categories: Gitlab
---

I got a weird error while converging gitlab, which looks like:

```console
RestClient::UnprocessableEntity
-------------------------------
422 Unprocessable Entity
```
So, I have to enable verbose logging for RestClient to see what's happening.
```ruby
RestClient.log = Logger.new(STDOUT)
```
So, the verbose log looks like:
```output
RestClient.post "https://localhost/api/v4/projects/32/hooks?private_token=xxxxx-xxx-xxxxx", "url=http%3A%2F%2Flocalhost%3A1234&push_events=true&issue_events=false&merge_requests_events=true&tag_push_events=false", "Accept"=>"application/json", "Accept-Encoding"=>"gzip, deflate", "Content-Length"=>"118", "Content-Type"=>"application/x-www-form-urlencoded", "User-Agent"=>"rest-client/2.0.2 (linux-gnu x86_64) ruby/2.4.2p198"
#=> 422 UnprocessableEntity | application/json 29 bytes
```
When I replay the request with `postman`, the error is actually like:
```json
{
    "error": "Invalid url given"
}
```
I look up and down to find out what is so-called `Invalid url` for hooks of gitlab, it leds me to link [Webhooks and insecure internal web services][Webhooks]

So, in a word, for security reason, gitlab has disallowed any addresses like 127.0.0.1, ::1 and 0.0.0.0, as well as IPv4 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 and IPv6 site-local (ffc0::/10) for hooks address.

Well, security is none of our concern, as our gitlab is hosted in private network with authorized personnel.

Currently, our webhook server is hosted as same as gitlab server, that is `http://localhost:1234`, we are no planning to move elsewhere. Luckily, gitlab also provides way to override this setting.

>This behavior can be overridden by enabling the option "Allow requests to the local network from hooks and services" in the "Outbound requests" section inside the Admin area under Settings

Cheers! let's do with api call [Application settings API][apis]

Here,it should do the trick for your:
```ruby
def allow_local_requests_for_hooks
  RestClient::Request.execute(method: :put, url: "https://localhost/api/v4/application/settings",headers: { params: { private_token: 'xxxxx-xxx-xxxxx-xxx', allow_local_requests_from_hooks_and_services: true }, accept: :json }, verify_ssl: false)
end

```
or one-time code
```bash
curl -X PUT --header "PRIVATE-TOKEN: XXXXX" 'https://localhost/api/v4/application/settings?allow_local_requests_from_hooks_and_services=true'
```
[Webhooks]: https://docs.gitlab.com/ee/security/webhooks.html
[apis]: https://docs.gitlab.com/ee/api/settings.html
