---
layout: post
title:  "Mysterious Affair For Mr. Jenkins"
date:   2019-04-20 07:35:50 +0800
categories: Jenkins
---

### Mr. Jenkins caught in trouble
Mr. Jenkins reports he can't finish his task recently because of that :

```sh
Mixlib::ShellOut::ShellCommandFailed: jenkins_script[enable protection] ( line 128) had an error: Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0], but received '255'
---- Begin output of java -jar "/var/chef/cache/jenkins-cli.jar" -s http://localhost/jenkins -"ssh" -user "jenkins" -i "/home/jenkins/.ssh/id_rsa" groovy = ----
STDOUT: 
STDERR: org.apache.sshd.common.SshException: Failed to get operation result within specified timeout: 10000
	at org.apache.sshd.common.future.AbstractSshFuture.verifyResult(AbstractSshFuture.java:96)
	at org.apache.sshd.client.future.DefaultAuthFuture.verify(DefaultAuthFuture.java:40)
	at org.apache.sshd.client.future.DefaultAuthFuture.verify(DefaultAuthFuture.java:33)
	at hudson.cli.SSHCLI.sshConnection(SSHCLI.java:106)
	at hudson.cli.CLI._main(CLI.java:592)
	at hudson.cli.CLI.main(CLI.java:426)
```

The most mysterious part of his report is that it's not always happening, so it may happen, may not happen, retry may help, but not always.

### Interesting clue

But Mr. Jenkins also notice one clue of that, if he takes `HudsonPrivateSecurityRealm` for authentication, thing runs pretty fast.

On the contrary, if he chooses to take on `LDAPSecurityRealm`, thing tends to be unpredictable。


### Tracking the crime

So, as senior technician, he can tell by intuition that there must be something with LDAP.

After enabling verbose logging with following...

```ini
hudson.security = ALL
jenkins.security = ALL
org.acegisecurity.ldap = ALL
org.acegisecurity.providers.ldap = ALL
```

Sure enough, he can see ldap search is happening for every single task he ran !

```sh
Searching for user 'jenkins', with user search [ searchFilter: 'mail={0}', searchBase: 'ou=ldap, o=example.com', scope: subtreesearchTimeLimit: 0derefLinkFlag: false ]
Apr 20, 2019 11:53:18 AM FINE org.acegisecurity.ldap.DefaultInitialDirContextFactory connect
Creating InitialDirContext with environment {java.naming.provider.url=ldaps://ldap.example.com:636/, java.naming.factory.initial=com.sun.jndi.ldap.LdapCtxFactory, com.sun.jndi.ldap.connect.timeout=30000, com.sun.jndi.ldap.connect.pool=true,java.naming.referral=follow, com.sun.jndi.ldap.read.timeout=60000}
```

### Searching for evidence

So, it's likely the timeout is caused by ldap search, as he can see clearly the timeout of LDAP is much much longer than `10000`, which is claiming the timeout time in the previous report.

```conf
com.sun.jndi.ldap.connect.timeout=30000
com.sun.jndi.ldap.read.timeout=60000
```

He does one simple test to verify:

add one line to `/etc/hosts`, which is to force ldap search to fail quickly
```
127.0.0.1 ldap.example.com
```
Again, he is right, his task now is running very fast now !

### Cache come to rescue

Mr. Jenkins consults his problem with the senior members in his family, and they are glad to help with this [magic][cache].

that is to enable ldap cache to increase search performance !

If thing must happen in the groovy way, take this:

```groovy
// size - of the cache in terms of number of elements
// ttl - of the cache in terms of seconds
// cache 300 items as long as 20 minutes
def cacheconfig = new LDAPSecurityRealm.CacheConfiguration(300, 1200)

realm = new LDAPSecurityRealm(server, rootDN, userBase, userFilter, groupBase, groupFilter, memberStrategy, managerDN, managerPass, true, false, cacheconfig, null, displayName, mailAddress, IdStrategy.CASE_INSENSITIVE, IdStrategy.CASE_INSENSITIVE)
strategy = new hudson.security.GlobalMatrixAuthorizationStrategy()
```

### Mr. Jenkins is happy

Mr. Jenkins is happy with cache enabled. To compare, without cache:

```sh
time java -jar /tmp/kitchen/cache/jenkins-cli.jar -ssh -user jenkins -i /home/jenkins/.ssh/id_rsa -s http://localhost:8080/jenkins version
org.apache.sshd.common.util.security.AbstractSecurityProviderRegistrar getOrCreateProvider
INFO: getOrCreateProvider(EdDSA) created instance of net.i2p.crypto.eddsa.EdDSASecurityProvider
org.apache.sshd.common.SshException: Failed to get operation result within specified timeout: 10000
	at org.apache.sshd.common.future.AbstractSshFuture.verifyResult(AbstractSshFuture.java:106)
	at org.apache.sshd.client.future.DefaultAuthFuture.verify(DefaultAuthFuture.java:40)
	at org.apache.sshd.client.future.DefaultAuthFuture.verify(DefaultAuthFuture.java:33)
	at hudson.cli.SSHCLI.sshConnection(SSHCLI.java:109)
	at hudson.cli.CLI._main(CLI.java:608)
	at hudson.cli.CLI.main(CLI.java:427)

real	0m10.717s
user	0m1.334s
sys	0m0.135s
```

With cache:
```sh
time java -jar /tmp/kitchen/cache/jenkins-cli.jar -ssh -user jenkins -i /home/jenkins/.ssh/id_rsa -s http://localhost:8080/jenkins version
org.apache.sshd.common.util.security.AbstractSecurityProviderRegistrar getOrCreateProvider
INFO: getOrCreateProvider(EdDSA) created instance of net.i2p.crypto.eddsa.EdDSASecurityProvider
1.89.2.1.1

real	0m1.550s
user	0m1.188s
sys	0m0.121s
```

### Unfortunate Mr. Jenkins

Unfortunately, if the user `jenkins` doesn't exist over ldap side, ldap will try over and over again to query for that user, as there is nothing to `cache`, which will unfortunately timeout Mr. Jenkins' task again....


[cache]:https://support.cloudbees.com/hc/en-us/articles/235216188-The-log-in-with-LDAP-plugin-is-very-slow 
