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
only checkout files changed on filesystem
```sh
Loading library jenkins-shared-library@main
FSSCM.checkout /var/jenkins_home/jobs/test/workspace/jenkins-shared-library to /var/jenkins_home/jobs/JenkinsFile/workspace@libs/277d200e682d6310644351b261dcc177e58ee098cb099d7d682af7825c7d31c6
Modified file: vars/runAnsible.groovy
Modified file: vars/const.groovy

FSSCM.check completed in 15 milliseconds
```

## Plugins

- filesystem_scm
- workflow-cps-global-lib
