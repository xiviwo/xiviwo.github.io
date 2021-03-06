---
layout: post
title:  "How to make chefspec work with data bag"
date:   2018-11-21 08:21:50 +0800
categories: Chef 
---

## Data Bag Issue

Working on some chefspec test case against chef libraries, it looks sound and fine with most methods in the libraries, but it comes with the trouble while trying to make these codes work.

```ruby
  def self.password?(username)
    return false unless Chef::DataBag.list.include? 'users'
    return false unless Chef::DataBag.load('users').keys.include?(username)
    item = Chef::DataBagItem.load('users', username)
  end
```

It continuously try to connect to my current chef server, which is offline by the time of testing, and throw `ERROR: Error connecting to ...., retry 5/5`.

It definitely doesn't look right, as it's supposed to connect to local chefzero server with `local_mode => true`.

What I need to do is to redirect the `Chef::Config[:chef_server_url]` to something locally.

## Chef Configuration
So,if we list the chef configuration of current test case with `STDOUT.puts Chef::Config.inspect`, it clearly show it's using configuration from file `"/Users/user/.chef/knife.rb"`, which is configuration file for my local `knife` command.

```txt
{:log_level=>:auto,
 :log_location=>"#<IO:<STDOUT>>",
 :config_file=>"/Users/user/.chef/knife.rb",
 :node_name=>"node1",
 :client_key=>"/Users/user/.chef/node1.pem",
 :chef_server_url=>"https://chef.example.com/organizations/myorg",
 :ssl_verify_mode=>:verify_none,
 :http_proxy=>nil,
 :https_proxy=>nil,
 :ftp_proxy=>nil,
 :no_proxy=>nil,
 :validation_client_name=>"chef-validator",
 :local_mode=>false}
```

## First try, chefzero://localhost:1
Since, it share configuration with local `knife` command, I suppose what make `knife` works should make test case work too. So, let's see what chef server `knife` works against.

```console
#=> knife node list -z -VV
WARN: No cookbooks directory found at or above current directory.  Assuming /Users/user.
INFO: Using configuration from /Users/user/.chef/knife.rb
INFO: Started chef-zero at chefzero://localhost:1 with repository at /Users/user
....
```
So, we can easily tell from the output, it use `chefzero://localhost:1` with configuration file `/Users/user/.chef/knife.rb`

## Redirect chef_server_url

From [here][stub], we can make our version of redirect:
```ruby
before do
  allow(Chef::Config).to receive(:[]).and_call_original
  allow(Chef::Config).to receive(:[]).with(:chef_server_url).and_return('chefzero://localhost:1')
end
```
Aha, let's try!  Unfortunately, it doesn't work
```error
ChefZero::ServerNotFound:
  No socketless chef-zero server on given port 1
```

## Second try

Ok, I search for a lot of example of chefspec, but can't get it to work. 
Basically, nearly 100% of the example need recipe and converge for the test case to work. 
But, for my cookbook, it only provide libaries and resource, has absolutely no need to include a recipe.


So, I look into the source code of ChefSpec for help, it turns out that if `ChefSpec::ServerRunner` is initiated, a local `ChefZero` server will be created per test case, which can be told as below:

```ruby
module ChefSpec
  # Rather than create a ChefZero instance per test case, simply create one
  # ChefZero instance and reset it for every test case.
  class ZeroServer
```
More interesting is, there is a port parameter here.

```ruby
      @server = ChefZero::Server.new(
        # Set the log level from RSpec, defaulting to warn
        log_level:  RSpec.configuration.log_level || :warn,
        port: RSpec.configuration.server_runner_port,

        # Set the data store
        data_store: data_store(RSpec.configuration.server_runner_data_store),
      )
```
Another quick search, turn out it use port `8889`, the default one

```ruby
RSpec.configure do |config|
  unless ENV['CHEFSPEC_NO_INCLUDE']
    config.include(ChefSpec::API)
  end

  config.add_setting :berkshelf_options, default: {}
  config.add_setting :file_cache_path
  config.add_setting :cookbook_root
  config.add_setting :cookbook_path
  config.add_setting :role_path
  config.add_setting :environment_path
  config.add_setting :file_cache_path
  config.add_setting :log_level, default: :warn
  config.add_setting :path
  config.add_setting :platform
  config.add_setting :version
  config.add_setting :server_runner_data_store, default: :in_memory
  config.add_setting :server_runner_clear_cookbooks, default: true
  config.add_setting :server_runner_port, default: (8889..8899)
end
```

Then, if we update the redirection to, it works !
```ruby
allow(Chef::Config).to receive(:[]).with(:chef_server_url).and_return('chefzero://localhost:8889')
```

Need to turn on `:debug` log_level to show the following logs.

```ruby
RSpec.configure do |config|
  config.log_level = :debug
end
```

the logs looks like:
```console
[2018-11-20T21:38:34+08:00] DEBUG: GET /organizations/chef/data
[2018-11-20T21:38:34+08:00] DEBUG: #<ChefZero::RestRequest:0x00007fbee2d1c6c8
 @env=
  {"SCRIPT_NAME"=>"",
   "SERVER_NAME"=>"localhost",
   "REQUEST_METHOD"=>"GET",
   "PATH_INFO"=>"/data",
   "QUERY_STRING"=>nil,
   "SERVER_PORT"=>8889,
   "HTTP_HOST"=>"localhost:8889",
   "HTTP_X_OPS_SERVER_API_VERSION"=>"1",
   "rack.url_scheme"=>"chefzero",
   "rack.input"=>#<StringIO:0x00007fbee2d1c740>},
 @rest_base_prefix=["organizations", "chef"],
 @rest_path=["organizations", "chef", "data"]>

[2018-11-20T21:38:34+08:00] DEBUG: 
--- RESPONSE (200) ---
{

}
--- END RESPONSE ---
```

But, the test case is still failing, why ? As you can see from the debug output, there is empty databag data returning from server.

## Create mocked databag data
Luckily, we have guidance [here][databag], basically, you just have to instantiate a `ChefSpec::ServerRunner` and call `create_data_bag` to create whatever databag you want.

Example:
```ruby
ChefSpec::ServerRunner.new do |node, server|
  server.create_data_bag('my_data_bag', {
    'item_1' => {
      'password' => 'abc123'
    },
    'item_2' => {
      'password' => 'def456'
    }
  })
end
```

## Standard Way

Look like the official way is to use `stub_data_bag` and `stub_data_bag_item` as below with `ChefSpec::SoloRunner`, but of course `ChefSpec::ServerRunner` is also an option.
```ruby

let(:chef_run) { ChefSpec::SoloRunner.new(platform: 'redhat', version: '7.4') }
  let(:list_data) { { 'users' => 'chefzero://localhost:1/data/users' } }

  before do
    allow_any_instance_of(Chef::DataBag).to receive(:list).and_return(list_data)
    stub_data_bag('users').and_return(%w(test_user1 test_user2))
    stub_data_bag_item('users', 'test_user1').and_return(
      'id' => 'test_user1',
      'password' => 'test-password1'
    )
    stub_data_bag_item('users', 'test_user2').and_return(
      'id' => 'test_user2',
      'password' => 'test-password2'
    )
  end

  context 'When given user is "test_user1"' do
    let(:username) { 'test_user1' }
    it 'has password' do
      chef_run
      expect(MYUsers.password?(username)).to eq(true)
    end

    it 'get password "test-password1"' do
      expect(MYUsers.password(username)).to eq('test-password1')
    end
  end
```
## Make it share context 

Let's take one step further, to make the two as share_context

***spec_helper.rb***
```ruby

RSpec.shared_context 'chef_solo' do
  let(:chef_run) { ChefSpec::SoloRunner.new(platform: 'redhat', version: '7.4') }
  let(:list_data) { { 'users' => 'chefzero://localhost:1/data/users' } }

  before do
    allow_any_instance_of(Chef::DataBag).to receive(:list).and_return(list_data)
    stub_data_bag('users').and_return(%w(test_user1 test_user2))
    stub_data_bag_item('users', 'test_user1').and_return(
      'id' => 'test_user1',
      'password' => 'test-password1'
    )
    stub_data_bag_item('users', 'test_user2').and_return(
      'id' => 'test_user2',
      'password' => 'test-password2'
    )
  end
end

RSpec.shared_context 'chef_zero' do
  let(:chef_run) do
    ChefSpec::ServerRunner.new(platform: 'redhat', version: '7.4') do |node, server|
      server.create_data_bag('users', 'test_user1' => {
                               'id' => 'test_user1',
                               'password' => {
                                 'encrypted_data' => "izqydW26h2uUjh3QbCa9u2Pm8DA1ZFEDCJqqIA/AuTsi\n",
                                 'iv' => "wcRpfKnpGbiDffB1\n",
                                 'auth_tag' => "lZ3NeZgc3/y+8IYLKGCYUw==\n",
                                 'version' => 3,
                                 'cipher' => 'aes-256-gcm'
                               }
                             }, 'test_user2' => {
                               'id' => 'test_user2',
                               'password' => {
                                 'encrypted_data' => "5Vd/4wadmGGGEdOhftVVJDH3RgADmoMvZAUC+fYvuE1p\n",
                                 'iv' => "fZmIjzs8SC0MW2Ud\n",
                                 'auth_tag' => "421m8RQhGe3Lnpcr9B+BuA==\n",
                                 'version' => 3,
                                 'cipher' => 'aes-256-gcm'
                               }
                             })
    end
  end
end
```
## Include the context along with shared_examples

```ruby

describe MyUsers do

  shared_examples 'password test' do

    context 'When given user is "test_user1"' do
      let(:username) { 'test_user1' }
      it ' has password' do
        chef_run
        expect(MyUsers.password?(username)).to eq(true)
      end

      it 'get password "test-password1"' do
        chef_run
        expect(MyUsers.password(username)).to eq('test-password1')
      end
    end

    context 'When given user is "test_user2"' do
      let(:username) { 'test_user2' }
      it ' has password' do
        chef_run
        expect(MyUsers.password?(username)).to eq(true)
      end

      it 'get password "test-password2"' do
        chef_run
        expect(MyUsers.password(username)).to eq('test-password2')
      end
    end

    context 'When given user is "abc"' do
      let(:username) { 'abc' }
      it ' has no password' do
        chef_run
        expect(MyUsers.password?(username)).to eq(false)
      end
    end
    
  end

  describe 'When context is chef zero' do
    include_context 'chef_zero'
    it_behaves_like 'password test'
  end

  describe 'When context is chef solo' do
    include_context 'chef_solo'
    it_behaves_like 'password test'
  end
end
```

[stub]: https://github.com/chefspec/chefspec/issues/64
[databag]: https://www.rubydoc.info/github/chefspec/chefspec