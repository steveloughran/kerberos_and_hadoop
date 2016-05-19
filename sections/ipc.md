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


> Man's respect for the imponderables varies according to his mental constitution and environment. Through certain modes of thought and training it can be elevated tremendously, yet there is always a limit.

> *[At the Root](https://en.wikisource.org/wiki/At_the_Root), HP Lovecraft, 1918.*

# Hadoop IPC Security

The Hadoop IPC system handles Kerberos ticket and Hadoop token authentication automatically.

1. The identity of the principals of services are configured in the hadoop site configuration
files.
1. Every IPC services uses java annotations, a metadata resource file and a custom `SecurityInfo`
subclass to define the security information of the IPC, including the key in the configuration
used to define the principal.
1. If a caller making a connection has a valid token (auth or delegate) it is used
to authenticate with the remote principal.
1. If a caller lacks a token, the Hadoop ticket will be used to acquire an authentication
token.
1. Applications may explicitly request delegation tokens to forward to other processes.
1. Delegation tokens are renewed in a background thread (which?).


## IPC authentication options

Hadoop IPC uses [SASL](sasl.html) to authenticate, sign and potentially encrypt
communications.

## Use Kerberos to authenticate sender and recipient

```xml
<property>
<name>hadoop.rpc.protection</name>
<value>authentication</value>
</property>
```

## Kerberos to authenticate sender and recipient, Checksums for tamper-protection

```xml
<property>
<name>hadoop.rpc.protection</name>
<value>integrity</value>
</property>
```

## Kerberos to authenticate sender and recipient, Wire Encryption

```xml
<property>
<name>hadoop.rpc.protection</name>
<value>privacy</value>
</property>
```





## Adding a new IPC interface to a Hadoop Service/Application

This is "fiddly". It's not impossible, it just involves effort.

In its favour: it's a lot easier than SPNEGO.

### Annotating a service interface

```java
@KerberosInfo(serverPrincipal = "my.kerberos.principal")
public interface MyRpc extends VersionedProtocol {
  long versionID = 0x01;
...
}
```

### `SecurityInfo` subclass

Every exported RPC service will need its own extension of the `SecurityInfo` class, to provide two things:

1. The name of the principal to use in this communication
1. The token used to authenticate ongoing communications.

### `PolicyProvider` subclass


```java
public class MyRpcPolicyProvider extends PolicyProvider {

  public Service[] getServices() {
    return  new Service[] {
     new Service("my.protocol.acl", MyRpc.class)
    };
  }

}
```

 This is used to inform the RPC infrastructure of the ACL policy: who may talk to the service. It must be explicitly passed to the RPC server

```java
rpcService.getServer() .refreshServiceAcl(serviceConf, new MyRpcPolicyProvider());
```

In practise, the ACL list is usually configured with a list of groups, rather than a user.

### `SecurityInfo` class 

```
public class MyRpcSecurityInfo extends SecurityInfo { ... }

```

### `SecurityInfo` resource file

The resource file `META-INF/services/org.apache.hadoop.security.SecurityInfo` lists all RPC APIs which have a matching SecurityInfo subclass in that JAR.

    org.example.rpc.MyRpcSecurityInfo

The RPC framework will read this file and build up the security information for the APIs (server side? Client side? both?)


### Authenticating a caller

How does an IPC endpoint validate the caller? If security is turned on,
the client will have had to authenticate with Kerberos, ensuring that
the server can determine the identity of the principal. 

This is something it can ask for when handling the RPC Call:

```java
UserGroupInformation callerUGI;

// #1: get the current user identity
try {
  callerUGI = UserGroupInformation.getCurrentUser();
} catch (IOException ie) {
  LOG.info("Error getting UGI ", ie);
  AuditLogger.logFailure("UNKNOWN", "Error getting UGI");
  throw RPCUtil.getRemoteException(ie);
}
```

The `callerUGI` variable is now set to the identity of the caller. If the caller
has delegated authority (tickets, tokens) then they still authenticate as
that principal they were acting as (possibly via a `doAs()` call).


```java
// #2 verify their permissions
String user = callerUGI.getShortUserName();
if (!checkAccess(callerUGI, MODIFY)) {
  AuditLog.unauthorized(user,
    KILL_CONTAINER_REQUEST,
    "User doesn't have permissions to " + MODIFY);
  throw RPCUtil.getRemoteException(new AccessControlException(
    + user + " lacks access "
    + MODIFY_APP.name()));
}
AuditLog.authorized(user, KILL_CONTAINER_REQUEST)
```

In ths example, there's a check to see if the caller can make a request which modifies
something in the service, if not the calls is rejected.

Note how failures are logged to an audit log; successful operations should be logged too.
The purpose of the audit log is determine the actions of a principal —both successful
and unsuccessful.

### Downgrading to unauthed IPC

IPC can be set up on the client to fall back to unauthenticated IPC if it can't negotiate
a kerberized connection. While convenient, this opens up some security vulnerabilitie -hence
the feature is generally disabled on secure clusters. It can/should be enabled when needed

```
-D ipc.client.fallback-to-simple-auth-allowed=true
```

As an example, this is the option on the command line for DistCp to copy from a secure cluster
to an insecure cluster, the destination only supporting simple authentication.

```
hadoop distcp -D ipc.client.fallback-to-simple-auth-allowed=true hdfs://secure:8020/lovecraft/books hdfs://insecure:8020/lovecraft/books
```

Although you can set it in a core-site.xml, this is dangerous from a security perpective

```xml
<property>
  <name>ipc.client.fallback-to-simple-auth-allowed</name>
  <value>true</value> 
</property>
```

*warning* it's tempting to turn this on during development, as it makes problems go away. As it is
not recommended in production: avoid except on the CLI during attempts to debug problems.
