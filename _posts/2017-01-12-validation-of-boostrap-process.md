---
layout: post
title:  "Chef Validator Bootstrap Process"
date:   2017-01-12 09:37:50 +0800
categories: Chef
---

### Validatorless Bootstrap

#### Run against Chef Zero

In last post regarding chef bootstrap, [here]({% post_url 2017-01-11-chef-bootstrap-process-explanation-with-code %}), a `chef-zero` server is running as chef server, and everything looks fine. 

What if we run against a real chef server, let's see what will happen.

Oooops, it's not working ....

```console
#=> k.client_builder.run
Creating new client for one_node
Net::HTTPServerException: 403 "Forbidden"
....
```

If we run equivalent `knife bootstrap` command, it will show you more friendly message.

The `client_key` or `client` the bootstrap command tries to act as has no create permission on the chef server, that is why we got `403 "Forbidden"` exeception at first place.

```console
#=> knife bootstrap localhost -N test_node -y --node-ssl-verify-mode none --server-url 'https://localhost/organizations/test_org'
Creating new client for test_node
ERROR: You authenticated successfully to https://localhost/organizations/test_org as bootstrap_node but you are not authorized for this action.
Response:  missing create permission
```
It makes perfect sense, as `Chef Zero` is designed to be that, since it has no `validation`, `authentication` or `authorization`, as: 

>Chef Zero is a simple, easy-install, in-memory Chef server that can be useful for Chef Client testing and chef-solo-like tasks that require a full Chef Server..... 
>It does NO input validation, authentication or authorization (it will not throw a 400, 401 or 403). 

#### Run against Real Chef server


We need chef `client` that has create permission, let's have a try with chef admin, or `root`.

Let's update the `Chef::Config` as below

```ruby
Chef::Config.node_name = 'root'
Chef::Config.client_key = "/etc/opscode/users/root/root.pem"
```

So, it works !

```console
#=> k.client_builder.run
Creating new client for one_node
Creating new node for one_node
=> #<Chef::Node:0x0000000002f01d40 ........> 
```

Look back the `create_client` code, it's self-explanatory: `REST client using the cli user's knife credentials`

```ruby
# Create the client object and save it to the Chef API
def create_client!
  Chef::ApiClient::Registration.new(node_name, client_path, http_api: rest).run
end
# @return [Chef::ServerAPI] REST client using the cli user's knife credentials
# this uses the users's credentials
def rest
  @rest ||= Chef::ServerAPI.new(chef_server_url)
end
```
So, what we have been discussing is known as a `validatorless bootstrap`, let's move to `Validator Bootstrap`.

### Validator Bootstrap

So, let's begin with so-called `Validation Client` `testorg-validator`, reset key of `testorg-validator` and download the `testorg-validator.pem` 

```ruby
Chef::Config.validation_client_name = 'testorg-validator'
Chef::Config.validation_key = "#{Dir.home}/.chef/testorg-validator.pem"
```

With `validation_key`, `Chef::Knife::Bootstrap` no longer calls `client_builder.run`, instead it call `knife_ssh.run` directly, refer to below:

```ruby
def run
  # ....
  if chef_vault_handler.doing_chef_vault? ||
      (Chef::Config[:validation_key] && !File.exist?(File.expand_path(Chef::Config[:validation_key])))
    client_builder.run

    chef_vault_handler.run(client_builder.client)

    bootstrap_context.client_pem = client_builder.client_path
  else
    ui.info("Doing old-style registration with the validation key at #{Chef::Config[:validation_key]}...")
    ui.info("Delete your validation key in order to use your user credentials instead")
    ui.info("")
  end

  ui.info("Connecting to #{ui.color(server_name, :bold)}")

  begin
    knife_ssh.run
  rescue Net::SSH::AuthenticationFailed
    # .....
  end
end
```

The `testorg-validator.pem` is passed on by the template/script, and cast into `/etc/chef/validation.pem` in remote node.


```console
#=> puts k.render_template
#.....
mkdir -p /etc/chef

cat > /etc/chef/validation.pem <<'EOP'
-----BEGIN RSA PRIVATE KEY-----
# ......
-----END RSA PRIVATE KEY-----

EOP
# ......
chmod 0600 /etc/chef/validation.pem

cat > /etc/chef/client.rb <<'EOP'
chef_server_url  "https://localhost/organizations/testorg"
validation_client_name "testorg-validator"
log_location   STDOUT
node_name "test_validator_node"
ssl_verify_mode :verify_none
EOP
# ......

```

At last, `chef-client` will be responsible for creating `client` and `node` using the validator key at `/etc/chef/validation.pem`, take chef-run below as example

```console
#=> k.run
Doing old-style registration with the validation key at /root/.chef/testorg-validator.pem...
Delete your validation key in order to use your user credentials instead

Connecting to localhost
-----> Existing Chef installation detected
Starting the first Chef Client run...
Starting Chef Client, version 14.9.19
Creating a new client identity for test_validator_node using the validator key.
resolving cookbooks for run list: []
Synchronizing Cookbooks:
Installing Cookbook Gems:
Compiling Cookbooks...
[2018-12-10T04:23:51+00:00] WARN: Node test_validator_node has an empty run list.
Converging 0 resources

Running handlers:
Running handlers complete
Chef Client finished, 0/0 resources updated in 03 seconds
```

#### Simple Simulation of Validator Bootstrap

We can use a simple script to demonstrate the validator bootstrap process.

NOTE: we are assuming `chef-client` or `chefdk` has been installed on your machine.

Start a dummy chef zero server in debug mode, so you can see clearly what's happening behind the scene.

```console
#=> sudo /opt/chefdk/embedded/bin/chef-zero -l debug --ssl -p 443
```

Save and run this script
```sh
# simple_validator_bootstrap.sh

mkdir -p /etc/chef

rm -fv /etc/chef/client.pem

cat > /etc/chef/validation.pem <<'EOP'
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEAwOkhMXYsEgb54muxnrHe4ItYS50xGAkRE5R2pGCLAU2sxPj5
5OlWMDUtmtkyFjIAxXE2jdxSSEmrvSrlAkn6ec+mGJGzVjd6Fkyuoep6FXSamqUi
860bT1KXA3S6+gJIBhvW/lwLaUL1erVT6VQbmOh3YgUzaaoOM4FFy+pski9adIJC
FZf/lHBJsXYEX3Da4tNSLsADU/gNpgK0rG1JdnvvoMaVp9khsc2YrBFpT4ugXoqY
7V2qKHTEmvBlQ8ldCnI4Vl/Ifmpqhtwllxn3YovIKfHNXwlYJ6v/EqYp1oqEJrjZ
72797djhUq5ENGt9EhQw1GQbuFbrqve/bnpLtQIDAQABAoIBAQCbtE+fXZNqpYjR
0TzXKxgKw03sEh9LGB5ZYG52dJod3jUB8ze4JQH0/ScnIgHEWm22748p51fekt/0
WofnVhC+evTERe2rPDHlh9U4SUpqwOf8xCc26VTurGnJV1GHc4nwrE3WljJ7rpj2
hx8IaGOyAohBbJM3yROXTNMqKLejMP/JByDCZ25UOkKhTXDLriWDmoWA9asqwk6v
U8zNKjse+bHcM6RRp96kQ31iwsv1xFtsrlm5aeOP9p6hI1VTIh/yoa5efLx/f50Z
IFIZ5kOKFP2mdsqAJ+adi8A40AAIbuHgG6xRgYVVzOlsX6TIpnZLxgb5+po1Lm+B
sBYNq6bhAoGBAO4WaLWedVXCTz5mV4AXFLOZW2aoYY7louD/sjfuE7Naj3XVVnFh
hvXNy24ig1q8+mNLk3tpt08BH4/G1fYEZ6MxM8OuTxls05liNIflLV+cyhlIOnxQ
U6Ncfa+XglyF6aBF2eln6Pwe338Is477wl9Zs1uA4BxgvUgKERmTmAApAoGBAM9s
m/yIhK+P7iWUUpNlaFG7P0Mq55bEDdxgiOgZti5RchkFzRaioWdiFYa3QlfDiaO1
COJinxjHggwTUrPIna2wGXsvLmih0rCFTyC045vDc0rtaAoXBHZIh4oIVD9BmhlE
0z67ROkTnOhr+iaMdjpf0564l7LRz7GxGKHoNrCtAoGAcBawuUCesP9H23LHIxC6
uEss0snXFDVcV11KBDbbo4axH6KOjdaCeVqnuXQaLy/lGbZM+r8sg89dkozj0m0E
dboGSsvXhXrMq9umK4xjri3cn8Z3cmtG1RQIQBCuWOzaro/0JYS8FWZbhi0Mi/ZO
7iEG5b9owzNwKWhD4Kyx1PECgYEAuFN83tJ2jwkpiU2ggAmKxa7PTiIPgYQiCSfk
IdXPdqO78A6erTHCmvunw3qRQyqp4sfa6ErZtQx+PbriMI/jx1iJnFVWOXcsot8k
bR0yctYiW4BThzvjJDXZ9MjoDPqANVpbGxER8MoUEtr5hk4mNkO37AGAFVGr7u1A
xYh1KVUCgYEAtfN5IsEZqxsqkvK9nCgLAbrwITqOQkL/LuwGy2ODwtijNT3PPZbe
D/4ShleDvvbBZRSath8tAXIm2ZfRUxt3xBJQv8WtdOexmfIO3sBPF8IT0N5ZbjUL
Y9m/UmPUvgENzLvzMrPJchTwLNg/Ep1VuQBEMIAsvqCLQqaLng1UTc8=
-----END RSA PRIVATE KEY-----

EOP
chmod 0600 /etc/chef/validation.pem




cat > /etc/chef/client.rb <<'EOP'
chef_server_url  "https://localhost:443"
validation_client_name "testorg-validator"
log_location   STDOUT
node_name "test_validator_node"
ssl_verify_mode :verify_none

EOP

cat > /etc/chef/first-boot.json <<'EOP'
{"run_list":[]}
EOP


echo "Starting the first Chef Client run..."

chef-client -j /etc/chef/first-boot.json
```

Sample output:
```console
Creating a new client identity for test_validator_node using the validator key.
[2018-12-10T14:59:09+08:00] INFO: Client key /etc/chef/client.pem is not present - registering
```