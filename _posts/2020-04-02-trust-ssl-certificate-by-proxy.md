---
layout: post
title:  "Trust SSL Certificate Issued by Interception Proxy"
date:   2020-04-02 16:25:50 +0800
categories: SSL
---

### S3 upload doesn't work all of a sudden 
So, in the morning, we have a incident reported that our java app based on AWS S3 suddenly ceases to work with error:


```
sun.security.validator.ValidatorException: PKIX path building failed: 
sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
unable to find valid certification path to requested target
```

At first glance, it looks like a certificate issue, according to this wonderful [post][atlassian] here, it generally means the java app doesn't trust the the certificate of S3. 

That doesn't make sense to me, S3 ceases to work due to certificate issue, that is less unlikely

### Proxy to blame

So, all our application are behind proxy, if we run curl command against s3, it has to be something like: 

```sh
=> curl -v --proxy myproxy:8999 https://s3.ap-southeast-1.amazonaws.com
...
Peer's certificate issuer is not recognized: 'CN=AAA SSL interception Service,OU=....'
...
```
We can tell from this error: our proxy intercept the SSL connection between our java app and S3, and rewrite the issuer with company internal root CA.


### Show the actual certificate behind the proxy

So, if we actuall want to find out what is certificate like via proxy, generally we could :

```sh
=> openssl s_client -showcerts -connect s3.ap-southeast-1.amazonaws.com:443 -proxy myproxy:8999 </dev/null
```

But,unfortunately,the openssl in our system doesn't support proxy switch yet.

Luckly, there is one alternative command available:

```sh
=> keytool -J-Dhttps.proxyHost=myproxy -J-Dhttps.proxyPort=8999 -printcert -rfc -sslserver s3.ap-southeast-1.amazonaws.com:443 > tmp.pem
```

And, we can show the content of this certificate with: 
```sh
=> openssl x509 -in tmp.pem -text -noout
....
        Issuer: C = XX, O = XXXXX, OU = XXXXXX, CN = AAA SSL interception Service
        Validity
            Not Before: Nov  9 00:00:00 2019 GMT
            Not After : Dec 10 12:00:00 2020 GMT
        Subject: C = US, ST = Washington, L = Seattle, O = "Amazon.com, Inc.", CN = *.s3-ap-southeast-1.amazonaws.com
....
```
Obviously, the s3 certificate is rewritten by our proxy with our internal root CA certificate, which is not trusted by our java program yet

### SSL test tool

There is a well-known java tool for testing validity  of trust stores, which is available [here][sslpoke] and we need the proxy version [here][sslpokeproxy], since everything is behind proxy in our case.

If we run the proxy version of the tool, we can exactly reproduce the error we met: 

```sh
=>java SSLPoke -Dhttps.proxyHost=myproxy -Dhttps.proxyPort=8999 s3.ap-southeast-1.amazonaws.com 443
sun.security.validator.ValidatorException: PKIX path building failed: 
sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
...
```
Famous sayings in computer world goes like: if issue can be reproduced, you are half way on the way to the problem. Hallelujah!

### Import the Internal root CA

The basic idea of fixing the issue is to import the internal root CA into our trust stores, so that the intercepted certificate can be trusted.

```bash
=> keytool -import  -keystore $JAVA_HOME/jre/lib/security/cacerts -file myrootca.crt -alias internalrootca
```

If we test again with the `SSLPoke`, it's now working. 
```bash
=>java SSLPoke -Dhttps.proxyHost=myproxy -Dhttps.proxyPort=8999 s3.ap-southeast-1.amazonaws.com 443
Successfully connected
```



[atlassian]:https://confluence.atlassian.com/kb/unable-to-connect-to-ssl-services-due-to-pkix-path-building-failed-error-779355358.html
[sslpoke]:https://gist.github.com/4ndrej/4547029
[sslpokeproxy]:https://gist.github.com/bric3/4ac8d5184fdc80c869c70444e591d3de
