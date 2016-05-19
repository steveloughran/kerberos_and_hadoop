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
  
# Zookeeper

Apache Zookeeper uses Kerberos + [SASL](sasl.md) to authenticate callers. 
The specifics are covered in [Zookeeper and SASL](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zookeeper+and+SASL)

Other than SASL, its access control is all based around secrets "Digests" which are shared between client and server, and sent over the (unencrypted) channel.
The Digest is stored in the ZK node; any client which provides the same Digest is considered to be that principal, and so gains
those rights. They cannot be relied upon to securely identify callers.

What's generally more troublesome about ZK is that its ACL-based permission scheme is "confusing".
Most developers are used to hierarchial permissions, in which the permissions of the parent
paths propagate down. If a root directory is world writeable, then anyone with those permissions
can implicity manipulate entries below it.

ZK nodes are not like this: permissions are there on a zknode-by-zknode basis.

For extra fun, ZK does not have any notion of groups; you can't have users in specific groups
(i.e. 'administrators')

The Hadoop Security book covers the client-side ZK APIs briefly.

## Enabling SASL in ZK

SASL is enabled in ZK by setting a system property. While adequate for a server,
it's less than convenient when using ZK in an application as it means something very important:
you cannot have a non-SASL and a SASL ZK connection at the same time.
Although you could create theoretically create one connection, change the system properties and then
create the next, Apache Curator, doesn't do this.

```java
System.setProperty("zookeeper.sasl.client", "true");
```

As everyone sensible uses Curator to handle transient disconnections and ZK node failover,
this isn't practicable. (Someone needs to fix this —volunteers welcome)

## Working with ZK

If you want to use ZK in production you have to

1. Remember that even in a secure cluster, parts of the ZK path, including the root `/` znode,
are usually world writeable. You may unintentionally be relying on this and be creating
insecure paths. (Some of our production tests explicitly check this, BTW).

1. Remember that ZK permissions have to be explicitly asked for: there is no inheritance. 
Set them for every node you intend to work with.

1. Lock down node permissions in a secure cluster, so that only authenticated users can 
read secret data or manipulate data which must only be written by specific services.
As an example, HBase and Accumulo both publish their binding information to ZK, The 
YARN Registry has per-user zknode paths set up so that all nodes under `/users/${username}`
are implicitly written by the user `$username`, so have their authority.

1. On an insecure cluster, do not try to create an ACL with "the current user" until the user
is actually authenticated. 

1. If you want administrative accounts to have access to znodes, explicitly set it.

## Basic code

```java
List<ACL> perms = new ArrayList<>();
if (UserGroupInformation.isSecurityEnabled()) {
  perms(new ACL(ZooDefs.Perms.ALL, ZooDefs.Ids.AUTH_IDS));
  perms.add(new ACL(ZooDefs.Perms.READ,ZooDefs.Ids.ANYONE_ID_UNSAFE));
} else {
  perms.add(new ACL(ZooDefs.Perms.ALL, ZooDefs.Ids.ANYONE_ID_UNSAFE));
}
zk.createPath(path, null, perms, CreateMode.PERSISTENT);
```


## Example YARN Registry

In the Hadoop yarn registry, in order to allow admin rights, we added a yarn
property to list principals who would be given full access.

To avoiding requiring all configuration files to list the explicit realm to use,
we added the concept that if the principal was listed purely as `user@`, rather than
`user@REALM`, we'd append the value of `hadoop.registry.kerberos.realm` —and
if that value was unset, the realm of the (logged in) caller.

This means that all users of the YARN registry in a secure cluster get znodes with
admin access to `yarn`, `mapred` and `hdfs` users in the current Kerberos Realm, unless
otherwise configured in the cluster's `core-site.xml`.

```xml
<property>
  <description>
    Key to set if the registry is secure. Turning it on
    changes the permissions policy from "open access"
    to restrictions on kerberos with the option of
    a user adding one or more auth key pairs down their
    own tree.
  </description>
  <name>hadoop.registry.secure</name>
  <value>false</value>
</property>

<property>
  <description>
    A comma separated list of Zookeeper ACL identifiers with
    system access to the registry in a secure cluster.

    These are given full access to all entries.

    If there is an "@" at the end of a SASL entry it
    instructs the registry client to append the default kerberos domain.
  </description>
  <name>hadoop.registry.system.acls</name>
  <value>sasl:yarn@, sasl:mapred@, sasl:hdfs@</value>
</property>

<property>
  <description>
    The kerberos realm: used to set the realm of
    system principals which do not declare their realm,
    and any other accounts that need the value.

    If empty, the default realm of the running process
    is used.

    If neither are known and the realm is needed, then the registry
    service/client will fail.
  </description>
  <name>hadoop.registry.kerberos.realm</name>
  <value></value>
</property>
```
 
## ZK Client and JAAS

Zookeeper needs a [jaas context](jaas.html) in SASL mode.

It will actually attempt to fallback to unauthorized if it doesn't get one

ZK's example JAAS client config

```
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=true
  keyTab="/path/to/client/keytab"
  storeKey=true
  useTicketCache=false
  principal="yourzookeeperclient";
};
```

And here is one which should work for using your login credentials instead

```
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=false
  useTicketCache=true
  principal="user@REALM";
  doNotPrompt=true
};
```

## How ZK reacts to authentication failures

The ZK server appears to react to a SASL authentication failure by closing the connection
_without sending any error back to the client_

This means that for a client, authentication problems surface as connection failures

```
2015-12-15 13:56:30,066 [main] DEBUG zk.CuratorService (zkList(695)) - ls /registry
Exception: `/': Failure of ls() on /: org.apache.zookeeper.KeeperException$ConnectionLossException: KeeperErrorCode = ConnectionLoss for /registry: KeeperErrorCode = ConnectionLoss for /registry
2015-12-15 13:56:58,892 [main] ERROR main.ServiceLauncher (error(344)) - Exception: `/': Failure of ls() on /: org.apache.zookeeper.KeeperException$ConnectionLossException: KeeperErrorCode = ConnectionLoss for /registry: KeeperErrorCode = ConnectionLoss for /registry
org.apache.hadoop.registry.client.exceptions.RegistryIOException: `/': Failure of ls() on /: org.apache.zookeeper.KeeperException$ConnectionLossException: KeeperErrorCode = ConnectionLoss for /registry: KeeperErrorCode = ConnectionLoss for /registry
  at org.apache.hadoop.registry.client.impl.zk.CuratorService.operationFailure(CuratorService.java:403)
  at org.apache.hadoop.registry.client.impl.zk.CuratorService.operationFailure(CuratorService.java:360)
  at org.apache.hadoop.registry.client.impl.zk.CuratorService.zkList(CuratorService.java:701)
  at org.apache.hadoop.registry.client.impl.zk.RegistryOperationsService.list(RegistryOperationsService.java:154)
  at org.apache.hadoop.registry.client.binding.RegistryUtils.statChildren(RegistryUtils.java:204)
  at org.apache.slider.client.SliderClient.actionResolve(SliderClient.java:3345)
  at org.apache.slider.client.SliderClient.exec(SliderClient.java:431)
  at org.apache.slider.client.SliderClient.runService(SliderClient.java:323)
  at org.apache.slider.core.main.ServiceLauncher.launchService(ServiceLauncher.java:188)
  at org.apache.slider.core.main.ServiceLauncher.launchServiceRobustly(ServiceLauncher.java:475)
  at org.apache.slider.core.main.ServiceLauncher.launchServiceAndExit(ServiceLauncher.java:403)
  at org.apache.slider.core.main.ServiceLauncher.serviceMain(ServiceLauncher.java:630)
  at org.apache.slider.Slider.main(Slider.java:49)
Caused by: org.apache.zookeeper.KeeperException$ConnectionLossException: KeeperErrorCode = ConnectionLoss for /registry
  at org.apache.zookeeper.KeeperException.create(KeeperException.java:99)
  at org.apache.zookeeper.KeeperException.create(KeeperException.java:51)
  at org.apache.zookeeper.ZooKeeper.getChildren(ZooKeeper.java:1590)
  at org.apache.curator.framework.imps.GetChildrenBuilderImpl$3.call(GetChildrenBuilderImpl.java:214)
  at org.apache.curator.framework.imps.GetChildrenBuilderImpl$3.call(GetChildrenBuilderImpl.java:203)
  at org.apache.curator.RetryLoop.callWithRetry(RetryLoop.java:107)
  at org.apache.curator.framework.imps.GetChildrenBuilderImpl.pathInForeground(GetChildrenBuilderImpl.java:200)
  at org.apache.curator.framework.imps.GetChildrenBuilderImpl.forPath(GetChildrenBuilderImpl.java:191)
  at org.apache.curator.framework.imps.GetChildrenBuilderImpl.forPath(GetChildrenBuilderImpl.java:38)
  at org.apache.hadoop.registry.client.impl.zk.CuratorService.zkList(CuratorService.java:698)
  ... 10 more
```

If you can telnet into the ZK host & port then ZK is up, but rejecting authenticated calls.

You need to go to the server logs (e.g. `/var/log/zookeeper/zookeeper.out`) to see what actually went wrong:

```
2015-12-15 13:56:30,995 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@197] - Accepted socket connection from /192.168.56.1:55882
2015-12-15 13:56:31,004 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@868] - Client attempting to establish new session at /192.168.56.1:55882
2015-12-15 13:56:31,031 - INFO  [SyncThread:0:ZooKeeperServer@617] - Established session 0x151a5e1345d0003 with negotiated timeout 40000 for client /192.168.56.1:55882
2015-12-15 13:56:31,181 - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@969] - Client failed to SASL authenticate: javax.security.sasl.SaslException:
 GSS initiate failed [Caused by GSSException: Failure unspecified at GSS-API level (Mechanism level: Specified version of key is not available (44))]
2015-12-15 13:56:31,181 - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@975] - Closing client connection due to SASL authentication failure.
2015-12-15 13:56:31,182 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@1007] - Closed socket connection for client /192.168.56.1:55882 which had
 sessionid 0x151a5e1345d0003
2015-12-15 13:56:31,182 - ERROR [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@178] - Unexpected Exception: 
java.nio.channels.CancelledKeyException
        at sun.nio.ch.SelectionKeyImpl.ensureValid(SelectionKeyImpl.java:73)
        at sun.nio.ch.SelectionKeyImpl.interestOps(SelectionKeyImpl.java:77)
        at org.apache.zookeeper.server.NIOServerCnxn.sendBuffer(NIOServerCnxn.java:151)
        at org.apache.zookeeper.server.NIOServerCnxn.sendResponse(NIOServerCnxn.java:1081)
        at org.apache.zookeeper.server.ZooKeeperServer.processPacket(ZooKeeperServer.java:936)
        at org.apache.zookeeper.server.NIOServerCnxn.readRequest(NIOServerCnxn.java:373)
        at org.apache.zookeeper.server.NIOServerCnxn.readPayload(NIOServerCnxn.java:200)
        at org.apache.zookeeper.server.NIOServerCnxn.doIO(NIOServerCnxn.java:244)
        at org.apache.zookeeper.server.NIOServerCnxnFactory.run(NIOServerCnxnFactory.java:208)
        at java.lang.Thread.run(Thread.java:745)
2015-12-15 13:56:31,186 - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@346] - Exception causing close of session 0x151a5e1345d0003
 due to java.nio.channels.CancelledKeyException
2015-12-15 13:56:32,540 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@197] - Accepted socket connection from /192.168.56.1:55883
2015-12-15 13:56:32,542 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@861] - Client attempting to renew session 0x151a5e1345d0003 at /192.168.56.1:55883
2015-12-15 13:56:32,543 - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@617] - Established session 0x151a5e1345d0003 with negotiated timeout 40000 for
 clie nt /192.168.56.1:55883
2015-12-15 13:56:32,547 - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@969] - Client failed to SASL authenticate: javax.security.sasl.SaslException:
 GSS initiate failed [Caused by GSSException: Failure unspecified at GSS-API level (Mechanism level: Specified version of key is not available (44))]
```


## Troubleshooting ZK

There's a nice list from Jeremy Custenborder of what to do to troubleshoot ZK
on [ZOOKEEPER-2345](https://issues.apache.org/jira/browse/ZOOKEEPER-2345?focusedCommentId=15134725&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-15134725)

