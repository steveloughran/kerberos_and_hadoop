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

# Hadoop Service Tokens

> Man rules now where They ruled once;
> They shall soon rule where man rules now.
> After summer is winter, and after winter summer.
> They wait patient and potent, for here shall They reign again


Hadoop Service "Tokens" are the other side of the complexity of Kerberos and Hadoop;
close enough to be confusing, different enough that the confusion becomes
dangerous.


## Core Concepts

1. Hadoop tokens are issued by services, *for use by that service and its
distributed components*.
1. Tokens are obtained by clients from services, usually through some IPC call
which returns a token.
1. Tokens can contain arbitrary data issued by a service and marshalled
into a byte array via the Hadoop `Writable` interfce.
1. This data is kept opaque to clients by encrypting the marshalled data with
a password. The encrypted data is given to the client, which can then resubmit it
to the service or peers *with the same password*. Here it can be decoded and used.
1. Therefore, provided the password is sufficiently complex, it should be impossible
for a client application to view *or tamper with* the data.
1. Delegation tokens *may* be passed to other services or other applications.
The services processing tokens do not (normally) validate the identity of the caller, merely
that the token is valid.
1. Tokens expire; they are invalid after that expiry time.
1. Tokens *may* be renewed before they expire, so returning a new token.
(Not all services support token renewal).
1. Token renewal may be repeated, until a final expiry time is reached, often 7 days.
1. Tokens may be revoked, after which they are not valid. This is possible if the 
(stateful) service maintains a table of valid tokens.


Here are some example uses of tokens

### HDFS access tokens in a launched YARN application

1. An application running in an account logged in to Kerberos requests a delegation token
(DT) from HDFS, (API call: `FileSystem.addDelegationTokens()`).
1. This token instance is added to the `ContainerLaunchContext`.
1. The application is launched.
1. The launched App master can retrieve the tokens it has been launched with. One
of these will be the delegation token.
1. The delegation token is used to authenticate the application, renewing it
as needed.
1. When the application completes, the token is revoked.
  (`ApplicationSubmissionContext.setCancelTokensWhenComplete`).
  
### Block Access tokens within HDFS

1. An authenticated client asks the NN for access to data.
1. The NN determines the block ID of the data, and returns a block token to the caller.
1. This contains encrypted information about the block, particularly its ID the access
rights granted to the caller.
1. The client application locates a DN hosting the block, and issues an access request
on the block, passing in the block token.
1. The DN (which has the same secret password as the NN), decrypts the token to validate
access permissions of the caller.
1. If valid, access to the data is granted.


### HBase Access token in Spark Job submission

The Spark submission client, if configured to do so, will ask HBase for a token. This
is added to the application launch context, so that a spark job may talk to HBase.


## Supporting Tokens in a Service


To support tokens you need to define your own token identifier, which is a marshallable
object containing all data to be contained within a token.


### Binding

Binding information must be declared in service metadata files, so the Java
ServiceLoader can find them. The Token Identifier class is declared in
`META-INF/services/org.apache.hadoop.security.token.TokenIdentifier`; 



Contents of `hadoop-tools/hadoop-azure/src/main/resources/META-INF/services/org.apache.hadoop.security.token.TokenIdentifier`

```
org.apache.hadoop.fs.azure.security.WasbDelegationTokenIdentifier
```

If a token is renewable, it must also provide a token renewer declaration
in in `META-INF/services/org.apache.hadoop.security.token.TokenRenewer`


Contents of `hadoop-tools/hadoop-azure/src/main/resources/META-INF/services/org.apache.hadoop.security.token.TokenRenewer`

```
org.apache.hadoop.fs.azure.security.WasbTokenRenewer
```

Important: make these classes fast to load, and resilient to not having dependencies
on the classpath. Why? They classes will be loaded whenever *any* token is decoded.
A slow loading class applications down; one which actually fails during
loading creates needless support calls.


### Token Service and Kind

A *token kind* is the string identifier used to uniquely identify the binding implementation
class; this class's implementation of `TokenIdentifier.getKind()` is used to look up
the implementation of `TokenIdentifier` registered. This string is saved as a `Text` entry
in the token file; when the token is decoded the implementation is located.

The token kind *must* be unique amongst all possible token implementation classes.

A *Token Service* is a reference to a service, used to locate it in Credentials.
`Credentials` persists tokens as a map from (service -> token), where the service
is that returned by `Token.getService()`. Generally this *must* be unique to
both the service *and the endpoint offering that service*. HDFS uses the IP address
of the NN host as part of its service identifier. 
See `SecurityUtil.buildDTServiceName()` for the algorithm here. It is not mandatory
to use this -any identifier unique for a service kind and installation is sufficient.
