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

# Tales of Terror

The following are all true stories. We welcome more submissions of these stories, especially
covering the steps taken to determine what was wrong.


## The Zookeeper's Birthday Present


A client program could not work with zookeeper: the connections were being broken. But it
was working for everything else.

The cluster was one year old that day.

It turns out that ZK reacts to an auth failure by logging something in its logs, and breaking
the client connection —without any notification to the client. Rather than a network problem
(initial hypothesis), this a Kerberos problem. How was that worked out? By examining the
Zookeeper logs —there was nothing client-side except the reports of connections being closed
and the ZK client attempting to retry.

When a Kerberos keytab is created, the entries in it have a lifespan. The default value is one
year. This was its first birthday, hence ZK wouldn't trust the client.

**Fix: create new keytabs, valid for another year, and distribute them.**

## The Principal With No Realm

This one showed up during release testing —credit to Andras Bokor for tracking it all down.

A stack trace

```
16/01/16 01:42:39 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
java.io.IOException: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "os-u14-2-2.novalocal/172.22.73.243"; destination host is: "os-u14-2-3.novalocal":8020; 
  at org.apache.hadoop.net.NetUtils.wrapException(NetUtils.java:773)
  at org.apache.hadoop.ipc.Client.call(Client.java:1431)
  at org.apache.hadoop.ipc.Client.call(Client.java:1358)
  at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:229)
  at com.sun.proxy.$Proxy11.getFileInfo(Unknown Source)
  at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.getFileInfo(ClientNamenodeProtocolTranslatorPB.java:771)
  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
  at java.lang.reflect.Method.invoke(Method.java:606)
  at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:252)
  at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:104)
  at com.sun.proxy.$Proxy12.getFileInfo(Unknown Source)
  at org.apache.hadoop.hdfs.DFSClient.getFileInfo(DFSClient.java:2116)
  at org.apache.hadoop.hdfs.DistributedFileSystem$22.doCall(DistributedFileSystem.java:1315)
  at org.apache.hadoop.hdfs.DistributedFileSystem$22.doCall(DistributedFileSystem.java:1311)
  at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
  at org.apache.hadoop.hdfs.DistributedFileSystem.getFileStatus(DistributedFileSystem.java:1311)
  at org.apache.hadoop.fs.FileSystem.exists(FileSystem.java:1424)
```

This looks like a normal "not logged in" problem, except for some little facts: 

1. The user was logged in.
1. The failure was replicable.
1. It only surfaced on OpenJDK, not oracle JDK.
1. Everything worked on OpenJDK 7u51, but not on OpenJDK 7u91.

Something had changed in the JDK to reject the login on this system (ubuntu, virtual test cluster).

`Kdiag` didn't throw up anything obvious. What did show some warning was `klist`:

```
Ticket cache: FILE:/tmp/krb5cc_2529
Default principal: qe@REALM

Valid starting       Expires              Service principal
01/16/2016 11:07:23  01/16/2016 21:07:23  krbtgt/REALM@REALM 
  renew until 01/23/2016 11:07:23
01/16/2016 13:13:11  01/16/2016 21:07:23  HTTP/hdfs-3-5@
  renew until 01/23/2016 11:07:23
01/16/2016 13:13:11  01/16/2016 21:07:23  HTTP/hdfs-3-5@REALM
  renew until 01/23/2016 11:07:23
```

See that? There's a principal which doesn't have a stated realm. Does that matter? 

In OracleJDK, and OpenJDK 7u51, apparently not. In OpenJDK 7u91, yes

There's some new code in `sun.security.krb5.PrincipalName`

```java
// Validate a nameStrings argument
private static void validateNameStrings(String[] ns) {
    if (ns == null) {
        throw new IllegalArgumentException("Null nameStrings not allowed");
    }
    if (ns.length == 0) {
        throw new IllegalArgumentException("Empty nameStrings not allowed");
    }
    for (String s: ns) {
        if (s == null) {
            throw new IllegalArgumentException("Null nameString not allowed");
        }
        if (s.isEmpty()) {
            throw new IllegalArgumentException("Empty nameString not allowed");
        }
    }
}
```

This checks the code, and rejects if nothing is valid. Now, how does something invalid get in?
Setting `HADOOP_JAAS_DEBUG=true` and logging at debug turned up output,

With 7u51:
```
16/01/20 15:13:20 DEBUG security.UserGroupInformation: using kerberos user:qe@REALM
```

With 7u91:

```
16/01/20 15:10:44 DEBUG security.UserGroupInformation: using kerberos user:null
```

Which means that the default principal wasn't being picked up, instead some JVM specific introspection
had kicked in —and it was finding the principal without a realm, rather than the one that was.


*Fix: add a `domain_realm` in `/etc/krb5.conf` mapping hostnames to realms *

```
[domain_realm]
  hdfs-3-5.novalocal = REALM
```

A `klist` then returns a list of credentials without this realm-less one in.

```
Valid starting       Expires              Service principal
01/17/2016 14:49:08  01/18/2016 00:49:08  krbtgt/REALM@REALM
  renew until 01/24/2016 14:49:08
01/17/2016 14:49:16  01/18/2016 00:49:08  HTTP/hdfs-3-5@REALM
  renew until 01/24/2016 14:49:08
```

Because this was a virtual cluster, DNS/RDNS probably wasn't working, presumably kerberos
didn't know what realm the host was in, and things went downhill. It just didn't show in
any validation operations, merely in the classic "no TGT" error.

## The AD realm redirection failure

Real-life example: 

* Company ACME has one ActiveDirectory domain per continent.
* Domains have mutual trust enabled.
* AD is also used for Kerberos authentication.
* Kerberos trust is handled by a few AD servers in the "root" domain.
* Hadoop cluster is running in Europe.

When a South American user opens a SSH session on the edge node, authentication is done by LDAP, no issue
the dirty work is done by the AD servers. 
But when a S.A. user tries to connect to HDFS or HiveServer2, with principal `Me@SAM.ACME.COM`,
then the Kerberos client must make several hops...

1. AD server for @SAM.ACME.COM says "no, can't create ticket for svc/somehost@EUR.ACME.COM"
1. AD server for @SAM.ACME.COM says "OK, I can get you a credential to @ACME.COM, see what they can do there" alas, 
1. There's no AD server defined in conf file for @ACME.COM
1. This leads to the all to familiar message, `Fail to create credential. (63) - No service creds`

Of course the only thing displayed in logs was the final error message.
Even after enabling the "secret" debug flags, it was not clear what the client was trying to do
with all these attempts.
But the tell-tale was the "following capath" comment on each hop, because `CAPATH` is actually
an optional section in `krb5.conf`. Fix: add the information about cross realm authentication
to the `krb5.conf` file.

(CAPATH coverage: [MIT](http://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-admin/capaths.html),
[Redhat](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Managing_Smart_Cards/Setting_Up_Cross_Realm_Authentication.html)
