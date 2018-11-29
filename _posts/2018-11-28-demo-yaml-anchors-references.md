---
layout: post
title:  "A demo of YAML anchors, references and nested values"
date:   2018-11-28 15:47:50 +0800
categories: yaml
---
Just a good example of yaml reuse from [here][gist]

```yaml
default: &DEFAULT
  URL:          stooges.com
  throw_pies?:  true  
  stooges:  &stooge_list
    larry:  first_stooge
    moe:    second_stooge
    curly:  third_stooge
  
development:
  <<: *DEFAULT
  URL:      stooges.local
  stooges: 
    shemp: fourth_stooge
    
test:
  <<: *DEFAULT
  URL:    test.stooges.qa
  stooges: 
    <<: *stooge_list
    shemp: fourth_stooge
```
Read again with ruby code:
```ruby
stooges = YAML::load( File.read('stooges.yml') )
puts stooges.to_yaml
```
So, the final result of previously example is 

```yaml
---
default:
  URL: stooges.com
  throw_pies?: true
  stooges:
    larry: first_stooge
    moe: second_stooge
    curly: third_stooge
development:
  URL: stooges.local
  throw_pies?: true
  stooges:
    shemp: fourth_stooge
test:
  URL: test.stooges.qa
  throw_pies?: true
  stooges:
    larry: first_stooge
    moe: second_stooge
    curly: third_stooge
    shemp: fourth_stooge
```

[gist]: https://gist.github.com/bowsersenior/979804
