---
layout: post
title:  "A Tour with IBM Cloud Object Store"
date:   2019-02-27 21:11:50 +0800
categories: Cloud S3 Objectstore
---

### Origin

So, we are planning to backup our gitlab data to cloud, and we choose IBM Cloud Object Storage, also called [IBM COS][cos].

Basically IBM COS is cloud storage similar to AWS S3, even the [SDK][sdk] is a fork of AWS SDK. 

So, if you are fimilar to AWS S3, it's easy for you to start with IBM COS.

### Demostration with Python SDK 





[cos]:https://cloud.ibm.com/docs/services/cloud-object-storage?topic=cloud-object-storage-about-ibm-cloud-object-storage#about-ibm-cloud-object-storage
[sdk]:https://github.com/ibm/ibm-cos-sdk-python/