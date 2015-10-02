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

Apache Zookeeper uses Kerberos + [SASL](sasl.md) to authenticate callers 

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

```
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

```
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
   
 
