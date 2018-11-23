---
layout: post
title:  "Attributes in .kitchen.yml is not loaded for chefspec"
date:   2018-11-23 09:22:50 +0800
categories: chefspec 
---

### Attributes from .kitchen.yml is not designed to load for chefspec 

In test-kitchen, you may notice the command involving during converge

```console
#=> sudo -E /opt/chef/bin/chef-client --local-mode --config /tmp/kitchen/client.rb --log_level auto --force-formatter --json-attributes /tmp/kitchen/dna.json --chef-zero-port 8889 
```

`/tmp/kitchen/dna.json` is json file containing attributes converted from `.kitchen.yml`, `--json-attributes /tmp/kitchen/dna.json` is how `kitchen` loads the attributes defined in `.kitchen.yml` and doing converge with `chef-client`.

So, it's also happening to chefspec? Will it load those attributes as well ? Absolutely NOT, as far as I know..

You probably will get error like this, as you have `nil` attribute defined like `default[:mycookbook][:myapp][:user] = nil `

```console
NoMethodError:
       undefined method `[]' for nil:NilClass
```

### Deliberately set up your attributes from .kitchen.yml before chefspec

Basically, you need to deliberately override/set the attribute with `ChefSpec::ServerRunner` as demonstrated below.

***users.rb***
```ruby
default[:mycookbook][:myapp][:user][:username] = nil 
default[:mycookbook][:myapp][:user][:group] = nil
```

***.kitchen.yml***
```yaml
    attributes:
      mycookbook:
        myapp:
          user:
            username: 'john'
            group:  'admin'
```

***user_spec.rb***
```ruby
require 'spec_helper'

describe 'mycookbook::users' do
  context 'When all attributes are default' do
    let(:chef_run) do
      runner = ChefSpec::ServerRunner.new(platform: 'redhat', version: '7.4') do |node, server|
      	################### Override here ################################
        node.override[:mycookbook][:myapp][:user][:username] = 'john'
        node.override[:mycookbook][:myapp][:user][:group] = 'admin'
      end
      runner.converge(described_recipe)
    end
    let(:node) { chef_run.node }

    it 'converges successfully' do
      expect { chef_run }.to_not raise_error
    end

    it 'get correct setup' do
      expect(node[:mycookbook][:myapp][:user][:username]).to eq('john')
    end

  end

end
```
