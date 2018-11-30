---
layout: post
title:  "Reuse test cases with shared_examples of serverspec"
date:   2018-11-30 18:21:50 +0800
categories: [Serverspec,Chef]
---

One simple demo on how to reuse serverspec test cases with shared_examples

```ruby
shared_examples 'certficate_check' do |certificate|

  describe "verify ssl certificate against #{certificate}" do
    describe command 'echo | openssl s_client -connect 0:443 2>&1' do
      its(:stdout) { should contain 'Verify return code: 0' }
    end

    describe command('echo | openssl s_client -connect 0:443 2>&1 | openssl x509 -noout -serial') do
      its(:stdout) { should eq `openssl x509 -in /etc/gitlab/ssl/#{certificate} -noout -serial` }
    end

    describe command("openssl x509 -in /etc/gitlab/ssl/#{certificate} -noout -text 2>&1") do
      its(:stdout) { should contain 'DNS:\*\.example.com' }
    end

    describe x509_certificate("/etc/gitlab/ssl/#{certificate}") do
      it { should be_certificate }
      it { should have_purpose 'SSL server' }
      it { should_not have_purpose 'SSL server CA' }
      its(:keylength) { should be 2048 }
      its(:validity_in_days) { should be > 90 }
    end

    crt_base = File.basename(certificate, File.extname(certificate))

    describe x509_private_key("/etc/gitlab/ssl/#{crt_base}.key") do
      it { should be_valid }
      it { should_not be_encrypted }
      it { should have_matching_certificate("/etc/gitlab/ssl/#{certificate}") }
    end
  end
end

describe 'certficate check' do
  include_examples 'certficate_check', 'example.com.crt'
end
```
sample output

```console
       certficate check
         verify ssl certificate against example.com.crt
           Command "echo | openssl s_client -connect 0:443 2>&1"
             stdout
               should contain "Verify return code: 0"
           Command "echo | openssl s_client -connect 0:443 2>&1 | openssl x509 -noout -serial"
             stdout
               should eq "serial=58B7D57DSFSB543SDB4B3E1\n"
           Command "openssl x509 -in /etc/gitlab/ssl/example.com.crt -noout -text 2>&1"
             stdout
               should contain "DNS:\\*\\.example.com"
           X509 certificate "/etc/gitlab/ssl/example.com.crt"
             should be certificate
             should have purpose "SSL server"
             should not have purpose "SSL server CA"
             keylength
               should equal 2048
             validity_in_days
               should be > 90
           X509 private key "/etc/gitlab/ssl/example.com.key"
             should be valid
             should not be encrypted
             should have matching certificate "/etc/gitlab/ssl/example.com.crt"
```