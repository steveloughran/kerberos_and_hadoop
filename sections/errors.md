<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
  
   http://www.apache.org/licenses/LICENSE-2.0
  
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

# Error Messages to Fear

> The oldest and strongest emotion of mankind is fear, and the oldest and strongest kind of fear is fear of the unknown.

> *[Supernatural Horror in Literature](https://en.wikisource.org/wiki/Supernatural_Horror_in_Literature), HP Lovecraft, 1927.*


Security error messages appear to take pride in providing limited information. In particular,
they are usually some generic `IOException` wrapping a generic security exception. There is some
text in the message, but it is often `Failure unspecified at GSS-API level`, which means
"something went wrong".

Generally a stack trace with UGI in it is a security problem, *though it can be a network problem
surfacing in the security code*.

The underlying causes of problems are usually the standard ones of distributed systems: networking
and configuration.


Some of the OS-level messages are covered in Oracle's Troubleshooting Kerberos docs.

* [Common Kerberos Error Messages (A-M)](http://docs.oracle.com/cd/E19253-01/816-4557/trouble-6/index.html)
* [Common Kerberos Error Messages (N-Z)](http://docs.oracle.com/cd/E19253-01/816-4557/trouble-27/index.html)

Here are some of the common ones seen in Hadoop stack traces *and what we think are possible causes*

That is: on one or more occasions, the listed cause was the one which, when corrected, made
the stack trace go away.


## `GSS initiate failed` —no further details provided

```
WARN  ipc.Client (Client.java:run(676)) - Couldn't setup connection for rm@EXAMPLE.COM to /172.22.97.127:8020
org.apache.hadoop.ipc.RemoteException(javax.security.sasl.SaslException): GSS initiate failed
  at org.apache.hadoop.security.SaslRpcClient.saslConnect(SaslRpcClient.java:375)
  at org.apache.hadoop.ipc.Client$Connection.setupSaslConnection(Client.java:558)
```

This is widely agreed to be one of the most useless of error messages you can see. The only
ones that are worse than this are those which disguise a Kerberos problem, such as when ZK
closes the connection rather than saying "it couldn't authenticate".

If you see this connection, work out which service it was trying to talk to —and look in its
logs instead. Maybe, just maybe, there will be more information there.


## `Server not found in Kerberos database (7)` or `service ticket not found in the subject`

* DNS is a mess and your machine does not know its own name.
* Your machine has a hostname, but the service principal is a `/_HOST` wildcard and the hostname
is not one there's an entry in the keytab for.

We've seen this in the stdout of a NN

```
TGS_REQ { ... }UNKNOWN_SERVER: authtime 0, hdfs@EXAMPLE.COM for krbtgt/NOVALOCAL@EXAMPLE.COM, Server not found in Kerberos database
```

## `No valid credentials provided (Mechanism level: Illegal key size)]`

Your JVM doesn't have the extended cryptography package and can't talk to the KDC.
Switch to openjdk or go to your JVM supplier (Oracle, IBM) and download the JCE
extension package, and install it in the hosts where you want Kerberos to work.

## `Encryption type AES256 CTS mode with HMAC SHA1-96 is not supported/enabled`

```
[javax.security.sasl.SaslException: GSS initiate failed
[Caused by GSSException: Failure unspecified at GSS-API level
(Mechanism level: Encryption type AES256 CTS mode with HMAC SHA1-96 is not supported/enabled)]]
```

This has surfaced [in the distant past](https://issues.apache.org/jira/browse/HADOOP-10629).

Assume it means the same as above: the JVM doesn't have the JCE JAR installed.

## `No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt`

This may appear in a stack trace starting with something like:

```
javax.security.sasl.SaslException:
GSS initiate failed [Caused by GSSException:
No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
```

It's very common, and essentially means "you weren't authenticated"

Possible causes:

1. You aren't logged in via `kinit`.
1. You have logged in with `kinit`, but the tickets you were issued with have expired.
1. Your process was issued with a ticket, which has now expired.
1. You did specify a keytab but it isn't there or is somehow otherwise invalid
1. You don't have the Java Cryptography Extensions installed.
1. The principal isn't in the same realm as the service, so a matching TGT cannot be found.
That is: you have a TGT, it's just for the wrong realm.
1. Your Active Directory tree has the same principal in more than one place in the tree.
1. Your cached ticket list has been contaminated with a realmless-ticket, and the JVM is now
unhappy. (See ["The Principal With No Realm"](terrors.html))
1. The program you are running may be trying to log in with a keytab, but it's got a Hadoop
`FileSystem` instance *before that login took place*. Even if the service is not logged in. that
FS instance is unauthenticated.

[Apache HBase](https://hbase.apache.org) also has an unintentionally aggravating addition to the
generic GSSException which is likely to further drive the user into madness.

```
ipc.AbstractRpcClient – SASL authentication failed. The most likely cause is missing or invalid credentials. Consider ‘kinit’.
javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
```


## `Failure unspecified at GSS-API level (Mechanism level: Checksum failed)`

One of the classics

1. The password is wrong. A `kinit` command doesn't send the password to the KDC —it sends some hashed things
to prove to the KDC that the caller has the password. If the password is wrong, so is the hash, hence
an error about checksums.
1. There was a keytab, but it didn't work: the JVM has fallen back to trying to log in as the user.
1. Your keytab contains an old version of the keytab credentials, and cannot parse the
information coming from the KDC, as it lacks the up to date credentials.
1. SPENGO/REST: Kerberos is very strict about hostnames and DNS; this can somehow trigger the problem.
[http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization](http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization);
1. SPENGO/REST: Java 8 behaves differently from Java 6 and 7  which can cause problems
[HADOOP-11628](https://issues.apache.org/jira/browse/HADOOP-11628).


## `javax.security.auth.login.LoginException: No password provided`

When this surfaces in a server log, it means the server couldn't log in as the user. That is,
there isn't an entry in the supplied keytab for that user and the system (obviously) doesn't
want to fall back to user-prompted password entry.
 
Some of the possible causes

* The wrong keytab was specified.
* The configuration key names used for specifying keytab or principal were wrong.
* There isn't an entry in the keytab for the user.
* The spelling of the principal is wrong.
* The hostname of the machine doesn't match that of a user in the keytab, so a match of `service/host`
fails.

Ideally, services list the keytab and username at fault here. In a less than ideal world —that is
the one we live in— things are sometimes less helpful

Here, for example, is a Zookeeper trace, saying it is the user `null` that is at fault.

```
2015-12-15 17:16:23,517 - WARN  [main:SaslServerCallbackHandler@105] - No password found for user: null
2015-12-15 17:16:23,536 - ERROR [main:ZooKeeperServerMain@63] - Unexpected exception, exiting abnormally
java.io.IOException: Could not configure server because SASL configuration did not allow the  ZooKeeper server to authenticate itself properly: javax.security.auth.login.LoginException: No password provided
        at org.apache.zookeeper.server.ServerCnxnFactory.configureSaslLogin(ServerCnxnFactory.java:207)
        at org.apache.zookeeper.server.NIOServerCnxnFactory.configure(NIOServerCnxnFactory.java:87)
        at org.apache.zookeeper.server.ZooKeeperServerMain.runFromConfig(ZooKeeperServerMain.java:111)
        at org.apache.zookeeper.server.ZooKeeperServerMain.initializeAndRun(ZooKeeperServerMain.java:86)
        at org.apache.zookeeper.server.ZooKeeperServerMain.main(ZooKeeperServerMain.java:52)
        at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:116)
        at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:78)

```

## `javax.security.auth.login.LoginException: Unable to obtain password from user`

Believed to be the same as the `No password provided`

```
Exception in thread "main" java.io.IOException: Login failure for alice@REALM from keytab /etc/security/keytabs/spark.headless.keytab:
  javax.security.auth.login.LoginException: Unable to obtain password from user
        at org.apache.hadoop.security.UserGroupInformation.loginUserFromKeytab(UserGroupInformation.java:962)
        at org.apache.spark.deploy.SparkSubmit$.prepareSubmitEnvironment(SparkSubmit.scala:564)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:154)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:121)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: javax.security.auth.login.LoginException: Unable to obtain password from user
        at com.sun.security.auth.module.Krb5LoginModule.promptForPass(Krb5LoginModule.java:856)
        at com.sun.security.auth.module.Krb5LoginModule.attemptAuthentication(Krb5LoginModule.java:719)
```

The JVM Kerberos code needs to have the password for the user to login to kerberos with,
but Hadoop has told it "don't ask for a password'", so the JVM raises an exception.

Root causes should be the same as for the other message.

## `failure to login using ticket cache file`

You aren't logged via `kinit`, the application isn't configured to use a keytab. So: no ticket,
no authentication, no access to cluster services. 

you can use `klist -v` to show your current ticket cache 

fix: log in with `kinit`

## `Clock skew too great`

```
GSSException: No valid credentials provided
(Mechanism level: Attempt to obtain new INITIATE credentials failed! (null))
 . . . Caused by: javax.security.auth.login.LoginException: Clock skew too great

GSSException: No valid credentials provided (Mechanism level: Clock skew too great (37) - PROCESS_TGS

kinit: krb5_get_init_creds: time skew (343) larger than max (300)
```

This comes from the clocks on the machines being too far out of sync. 

This can surface if you are doing Hadoop work on some VMs and have been suspending and resuming them;
they've lost track of when they are. Reboot them.

If it's a physical cluster, make sure that your NTP daemons are pointing at the same NTP server,
one that is actually reachable from the Hadoop cluster.
And that the timezone settings of all the hosts are consistent.



## `KDC has no support for encryption type`

This crops up on the MiniKDC if you are trying to be clever about encryption types. It doesn't support many.


## `GSSException: No valid credentials provided (Mechanism level: Fail to create credential. (63) - No service creds)`


Rarely seen. Switching kerberos to use TCP rather than UDP makes it go away

In `/etc/krb5.conf`:

```
[libdefaults]
  udp_preference_limit = 1
```

Note also UDP is a lot slower to time out.

## `Receive timed out`

Usually in a stack trace like

```
Caused by: java.net.SocketTimeoutException: Receive timed out
  at java.net.PlainDatagramSocketImpl.receive0(Native Method)
  at java.net.AbstractPlainDatagramSocketImpl.receive(AbstractPlainDatagramSocketImpl.java:146)
  at java.net.DatagramSocket.receive(DatagramSocket.java:816)
  at sun.security.krb5.internal.UDPClient.receive(NetClient.java:207)
  at sun.security.krb5.KdcComm$KdcCommunication.run(KdcComm.java:390)
  at sun.security.krb5.KdcComm$KdcCommunication.run(KdcComm.java:343)
  at java.security.AccessController.doPrivileged(Native Method)
  at sun.security.krb5.KdcComm.send(KdcComm.java:327)
  at sun.security.krb5.KdcComm.send(KdcComm.java:219)
  at sun.security.krb5.KdcComm.send(KdcComm.java:191)
  at sun.security.krb5.KrbAsReqBuilder.send(KrbAsReqBuilder.java:319)
  at sun.security.krb5.KrbAsReqBuilder.action(KrbAsReqBuilder.java:364)
  at com.sun.security.auth.module.Krb5LoginModule.attemptAuthentication(Krb5LoginModule.java:735)
```

This means the UDP socket awaiting a response from KDC eventually gave up.

- The hostname of the KDC is wrong
- The IP address of the KDC is wrong
- There's nothing at the far end listening for requests.
- A firewall on either client or server is blocking UDP packets

Kerberos waits ~90 seconds before timing out, which is a long time to notice there's a problem.

Switch to TCP —at the very least, it will fail faster.

## `javax.security.auth.login.LoginException: connect timed out`

Happens when the system is set up to use TCP as an authentication channel, and the far end
KDC didn't respond in time.

- The hostname of the KDC is wrong
- The IP address of the KDC is wrong
- There's nothing at the far end listening for requests.
- A firewall somewhere is blocking TCP connections

##  `GSSException: No valid credentials provided (Mechanism level: Connection reset)`

We've seen this triggered in Hadoop tests after the MiniKDC through an exception; its thread
exited and hence the Kerberos client got a connection error.

When you see this assume network connectivity problems, or something up at the KDC itself.


## `Principal not found`

The hostname is wrong (or there is more than one hostname listed with different IP addresses) and so a principal
of the form `user/_HOST@REALM` is coming back with the wrong host, and the KDC doesn't find it.

See the comments above about DNS for some more possibilities.

## `Defective token detected (Mechanism level: GSSHeader did not find the right tag)`


Seen during SPNEGO Authentication: the token supplied by the client is not accepted by the server.

This apparently surfaces in [Java 8 version 8u40](http://sourceforge.net/p/spnego/discussion/1003769/thread/700b6941/#cb84);
if Kerberos server doesn't support the first authentication mechanism which the client
offers, then the client fails. Workaround: don't use those versions of Java.

This is [now acknowledged by Oracle](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8080129) and
has been fixed in 8u60.


## `Specified version of key is not available (44)`

```
Client failed to SASL authenticate:
  javax.security.sasl.SaslException:
    GSS initiate failed [Caused by GSSException: Failure unspecified at GSS-API level
     (Mechanism level: Specified version of key is not available (44))]
```

The meaning of this message —or how to fix it— is a mystery to all.

There is [some tentative coverage in Stack Overflow](http://stackoverflow.com/questions/24511812/krbexception-specified-version-of-key-is-not-available-44)

One possibility is that the keys in your keytab have expired. Did you know that can happen? It does.
One day your cluster works happily. The next your client requests are failing, with this message
surfacing in the logs.

```
 klist -kt zk.service.keytab 
Keytab name: FILE:zk.service.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
```

One thing to see there is the version number in the KVNO table.

Oracle describe the JRE's handling of version numbers [in their bug database](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6984764).

From an account logged in to the system, you can look at the client's version number

```
$ kvno zookeeper/devix@COTHAM
zookeeper/devix@COTHAM: kvno = 1
```

*Recommended strategy*

Rebuild your keytabs.

1. Take a copy of your current keytab dir, for easy reverting.
1. Use `ktlist -kt` to list the entries in each keytab.
1. Use `ls -al` to record their user + group values + permissions.
1. In `kadmin.local`, re-export every key to the keytabs which needed it with `xst -norandkey`
1. Make sure the file owners and permissions are as before.
1. Restart everything.


### `kinit: Client not found in Kerberos database while getting initial credentials`

This is fun: it means that the user is not known.

Possible causes

1. The user isn't in the database.
1. You are trying to connect to a different KDC than the one you thought you were using.
1. You aren't who you thought you were.

### `SIMPLE authentication is not enabled. Available:[TOKEN]"`

This surfaces on RPC connections when the client is trying to use "SIMPLE" (i.e. unauthenticated)
RPC, when the service is set to only support Kerberos ("TOKEN")

in the client configuration, set `hadoop.security.authentication` to `kerberos`.

(There is a configuration option to tell clients that they can support downgrading to simple, but
as it shouldn't be used, this document doesn't list it.


### `Request is a replay (34))`

The destination thinks the caller is attempting some kind of replay attack

1. The KDC is seeing too many attempts by the caller to authenticate as a specific principal,
assumes some kind of attack and rejects the request. This can happen if you have too many processes/
nodes all sharing the same principal. Fix: make sure you have `service/_HOST@REALM` principals
for all the services, rather than simple `service@REALM` principals. 
 
1. The timestamps of the systems are out of sync, so it looks like an old token be re-issued.
Check them all, including that of the KDC, make sure NTP is working, etc, etc.

## `AuthenticationToken ignored`

This has been seen in the HTTP logs of Hadoop REST/Web UIs:

```
WARN org.apache.hadoop.security.authentication.server.AuthenticationFilter:
AuthenticationToken ignored:
org.apache.hadoop.security.authentication.util.SignerException: Invalid signature
```

This means that the caller did not have the credentials to talk to a Kerberos-secured channel.

1. The caller may not be logged in.
2. The caller may have been logged in, but its kerberos token has expired, so its authentication headers are not considered valid any more.
3. The time limit of a negotiated token for the HTTP connection has expired. Here the calling app is expected to recognise this, discard its old token and renegotiate a new one. If the calling app is a YARN hosted service, then something should have been refreshing the tokens for you.


## `Found unsupported keytype (8)`

Happens [when the keytype supported by the KDC isn't supported by the JVM](http://stackoverflow.com/questions/31372742/found-unsupported-keytype-8-for-nn-hadoop-kerberoshadoop-kerberos)

Generate a keytab with a supported key encryption type. 

### "User: MyUserName is not allowed to impersonate ProxyUser"

Your code has tried to create a proxy user and talk to a Hadoop service with it, but
you do not have the right to impersonate that user. Only specific Hadoop services (Oozie, YARN RM)
have the right to do what is the Hadoop equivalent of (`su $user`)

See [Proxy user - Superusers Acting On Behalf Of Other Users](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/Superusers.html)

## `Kerberos credential has expired`


Seen on an IBM JVM in [HADOOP-9969](https://issues.apache.org/jira/browse/HADOOP-9969)

```
javax.security.sasl.SaslException:
  Failure to initialize security context [Caused by org.ietf.jgss.GSSException, major code: 8, minor code: 0
  major string: Credential expired
  minor string: Kerberos credential has expired]
```


The kerberos ticket has expired and not been renewed.

Possible causes

- The renewer thread somehow failed to start.
- The renewer did start, but didn't try to renew in time.
- A JVM/Hadoop code incompatibility stopped renewing from working.
- Renewal failed for some other reason.
- The client was kinited in and the token expired.
- Your VM clock has jumped forward and the ticket now out of date without any renewal taking place.


## SASL `No common protection layer between client and server`

Not Kerberos, SASL itself

```
16/01/22 09:44:17 WARN Client: Exception encountered while connecting to the server : 
javax.security.sasl.SaslException: DIGEST-MD5: No common protection layer between client and server
  at com.sun.security.sasl.digest.DigestMD5Client.checkQopSupport(DigestMD5Client.java:418)
  at com.sun.security.sasl.digest.DigestMD5Client.evaluateChallenge(DigestMD5Client.java:221)
  at org.apache.hadoop.security.SaslRpcClient.saslConnect(SaslRpcClient.java:413)
  at org.apache.hadoop.ipc.Client$Connection.setupSaslConnection(Client.java:558)
  at org.apache.hadoop.ipc.Client$Connection.access$1800(Client.java:373)
  at org.apache.hadoop.ipc.Client$Connection$2.run(Client.java:727)
  at org.apache.hadoop.ipc.Client$Connection$2.run(Client.java:723)
  at java.security.AccessController.doPrivileged(Native Method)
  at javax.security.auth.Subject.doAs(Subject.java:422)
  at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1657)
  at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:722)
  at org.apache.hadoop.ipc.Client$Connection.access$2800(Client.java:373)
  at org.apache.hadoop.ipc.Client.getConnection(Client.java:1493)
  at org.apache.hadoop.ipc.Client.call(Client.java:1397)
  at org.apache.hadoop.ipc.Client.call(Client.java:1358)
  at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:229)
  at com.sun.proxy.$Proxy23.renewLease(Unknown Source)
  at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.renewLease(ClientNamenodeProtocolTranslatorPB.java:590)
  at sun.reflect.GeneratedMethodAccessor9.invoke(Unknown Source)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
  at java.lang.reflect.Method.invoke(Method.java:497)
```

## On windows: `No authority could be contacted for authentication`

Reported on windows clients, especially related to the Hive ODBC client. This is kerberos, just
someone else's library.

1. Make sure that your system is happy in the AD realm, etc. Do this first.
1. Make sure you've configured the ODBC driver [according to the instructions](http://hortonworks.com/wp-content/uploads/2013/04/Hortonworks-Hive-ODBC-Driver-User-Guide.pdf).

## During service startup `java.lang.RuntimeException: Could not resolve Kerberos principal name:` + `unknown error`

This something which can arise in the logs of a service. Here, for example, is a datanode failure.

```
Could not resolve Kerberos principal name: java.net.UnknownHostException: xubunty: xubunty: unknown error
```

This is *not* a kerberos problem. It is a network problem being misinterpreted as a Kerberos problem, purely
because it surfaces in security code which assumes that all failures must be Kerberos related.


```
2016-04-06 11:00:35,796 ERROR org.apache.hadoop.hdfs.server.datanode.DataNode: Exception in secureMain java.io.IOException: java.lang.RuntimeException: Could not resolve Kerberos principal name: java.net.UnknownHostException: xubunty: xubunty: unknown error
  at org.apache.hadoop.http.HttpServer2.<init>(HttpServer2.java:347)
  at org.apache.hadoop.http.HttpServer2.<init>(HttpServer2.java:114)
  at org.apache.hadoop.http.HttpServer2$Builder.build(HttpServer2.java:290)
  at org.apache.hadoop.hdfs.server.datanode.web.DatanodeHttpServer.<init>(DatanodeHttpServer.java:108)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.startInfoServer(DataNode.java:781)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.startDataNode(DataNode.java:1138)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.<init>(DataNode.java:432)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.makeInstance(DataNode.java:2423)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.instantiateDataNode(DataNode.java:2310)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.createDataNode(DataNode.java:2357)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.secureMain(DataNode.java:2538)
  at org.apache.hadoop.hdfs.server.datanode.DataNode.main(DataNode.java:2562)
Caused by: java.lang.RuntimeException: Could not resolve Kerberos principal name: java.net.UnknownHostException: xubunty: xubunty: unknown error
  at org.apache.hadoop.security.AuthenticationFilterInitializer.getFilterConfigMap(AuthenticationFilterInitializer.java:90)
  at org.apache.hadoop.http.HttpServer2.getFilterProperties(HttpServer2.java:455)
  at org.apache.hadoop.http.HttpServer2.constructSecretProvider(HttpServer2.java:445)
  at org.apache.hadoop.http.HttpServer2.<init>(HttpServer2.java:340)
  ... 11 more
Caused by: java.net.UnknownHostException: xubunty: xubunty: unknown error
  at java.net.InetAddress.getLocalHost(InetAddress.java:1505)
  at org.apache.hadoop.security.SecurityUtil.getLocalHostName(SecurityUtil.java:224)
  at org.apache.hadoop.security.SecurityUtil.replacePattern(SecurityUtil.java:192)
  at org.apache.hadoop.security.SecurityUtil.getServerPrincipal(SecurityUtil.java:147)
  at org.apache.hadoop.security.AuthenticationFilterInitializer.getFilterConfigMap(AuthenticationFilterInitializer.java:87)
  ... 14 more
Caused by: java.net.UnknownHostException: xubunty: unknown error
  at java.net.Inet4AddressImpl.lookupAllHostAddr(Native Method)
  at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:928)
  at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1323)
  at java.net.InetAddress.getLocalHost(InetAddress.java:1500)
  ... 18 more2016-04-06 11:00:35,799 INFO org.apache.hadoop.util.ExitUtil: Exiting with status 12016-04-06 11:00:35,806 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: SHUTDOWN_MSG:
{code}
```

The root cause here was that the host `xubunty` which a service was configured to start on did not have an entry in `/etc/hosts`, nor DNS support.
The attempt to look up the IP address of the local host failed.

The fix: add the short name of the host to `/etc/hosts`.

This example shows why errors reported as Kerberos problems,
be they from the Hadoop stack or in the OS/Java code underneath,
are not always Kerberos problems.
Kerberos is fussy about networking; the Hadoop services have to initialize Kerberos
before doing any other work. As a result, networking problems often surface first
in stack traces belonging to the security classes, wrapped with exception messages
implying a Kerberos problem. Always follow down to the innermost exception in a trace
as the immediate symptom of a problem, the layers above attempts to interpret that,
attempts which may or may not be correct.

## Against Active Directory: `Realm not local to KDC while getting initial credentials`

Nobody quite knows.

It's believed to be related to Active Directory cross-realm/forest stuff, but there
are hints that it can also be raised when the kerberos client is trying to auth
with a KDC, but supplying a hostname rather than the realm.

This may be because you have intentionally or unintentionally created [A Disjoint Namespace](https://technet.microsoft.com/en-us/library/cc731125(v=ws.10).aspx))

If you read that article, you will get the distinct impression that even the Microsoft
Active Directory team are scared of Disjoint Namespaces, and so are going to a lot of
effort to convince you not to go there. It may seem poignant that even the developers of
AD are scared of this, but consider that these are probably inheritors of the codebase,
not the original authors, and the final support line for when things don't work. Their
very position in the company means that they get the worst-of-the-worst Kerberos-related
problems. If they say "Don't go there", it'll be based on experience of fielding those
support calls *and from having seen the Active Directory source code.*


* [Kerberos and the Disjoint Namespace](http://www.networkworld.com/article/2347477/microsoft-subnet/kerberos-and-the-disjoint-namespace.htmla)
* [Kerberos Principal Name Canonicalization and Cross-Realm Referrals](https://tools.ietf.org/html/rfc6806.html)

## `Can't get Master Kerberos principal`

The application wants to talk to a service (HDFS, YARN, HBase, ...), but it cannot
determine the Kerberos principal which it should obtain a ticket for.
This usually means that the client configuration doesn't declare the principal in
the appropriate option —for example "dfs.namenode.kerberos.principal".

## `SIMPLE authentication is not enabled. Available:[TOKEN, KERBEROS]` 

This surfaces in the client:

`org.apache.hadoop.security.AccessControlException: SIMPLE authentication is not enabled. Available:[TOKEN, KERBEROS]`

The client doesn't think security is enabled; it's only trying to use "SIMPLE" authentication, —the caller is 
whoever they say they are. However, the server will only take Kerberos tickets or Hadoop delegation tokens which
were previously acquired by by a caller with a valid Kerberos ticket. It is rejecting
the authentication request.
