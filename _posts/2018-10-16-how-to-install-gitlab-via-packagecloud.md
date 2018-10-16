---
layout: post
title:  "How to Install Gitlab via PackageCloud Cookbook "
date:   2018-10-16 11:00:50 +0800
categories: Chef
---

Ok,it's time to upgrade your gitlab server, and it's `11.3.5` on [gitlab][gitlab.com] now. 

Previously, what we do is to download the rpm and upload the rpm to our repo, that smells a bit of lack of automation

And it tells me there are other alternatives for installing things, so why not trying new way? 

![Gitlab Installation Options]({{site.url}}/_images/gitlabinstall.jpg)

So,basically, the first thing is to add dependency to your `metadata.rb`

```ruby
cookbook "packagecloud"
```

Then, change/update your code to something this.
I am quite sure you want to stick your gitlab to specific version until next upgrade, so you had better to specify version for `yum_package` block.

```ruby
packagecloud_repo "gitlab/gitlab-ee" do
  type "rpm"
  base_url "https://packages.gitlab.com/"
end

yum_package 'install gitlab' do
  package_name 'gitlab-ee.x86_64'
  action :install
  allow_downgrade true
  version '11.3.5-ee.0.el7'
end
```

Let's see what we got, it works, cheers !
```bash
  * yum_package[install gitlab] action install[2018-10-16T03:14:57+00:00] INFO: Processing yum_package[install gitlab] action install 
       [2018-10-16T03:14:57+00:00] INFO: Processing yum_package[install gitlab] action install
       [2018-10-16T03:14:57+00:00] INFO: yum_package[install gitlab] installing gitlab-ee-11.3.5-ee.0.el7.x86_64 from gitlab_gitlab-ee repository
       [2018-10-16T03:14:57+00:00] INFO: yum_package[install gitlab] installing gitlab-ee-11.3.5-ee.0.el7.x86_64 from gitlab_gitlab-ee repository
       [2018-10-16T03:16:57+00:00] INFO: yum_package[install gitlab] installed gitlab-ee at 11.3.5-ee.0.el7
       [2018-10-16T03:16:57+00:00] INFO: yum_package[install gitlab] installed gitlab-ee at 11.3.5-ee.0.el7
       
           - install version 11.3.5-ee.0.el7 of package gitlab-ee
       [2018-10-16T03:16:57+00:00] INFO: Chef Run complete in 128.669866186 seconds
       [2018-10-16T03:16:57+00:00] INFO: Chef Run complete in 128.669866186 seconds
```

[gitlab.com]: https://packages.gitlab.com/gitlab/gitlab-ee