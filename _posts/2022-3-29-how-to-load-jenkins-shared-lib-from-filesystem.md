---
layout: post
title:  "How to Load Jenkins Shared Libraries from Filesytem"
date:   2022-03-29 21:50:50 +0800
categories: Jenkins
---

## Config
Load shared libraries directly from local folder, without any git commit or push 

```groovy 
library identifier: 'jenkins-shared-library@main', 
        retriever: legacySCM(
        filesystem(clearWorkspace: false, 
        copyHidden: false, 
        path: "/var/jenkins_home/jobs/test/workspace/jenkins-shared-library"
        ))
```
## Plugins

- filesystem_scm
- workflow-cps-global-lib
