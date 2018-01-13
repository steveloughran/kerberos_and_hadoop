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

# HDFS

> It seemed to be a sort of monster, or symbol representing a monster, of a form which only a diseased fancy could conceive. If I say that my somewhat extravagant imagination yielded simultaneous pictures of an octopus, a dragon, and a human caricature, I shall not be unfaithful to the spirit of the thing. A pulpy, tentacled head surmounted a grotesque and scaly body with rudimentary wings; but it was the general outline of the whole which made it most shockingly frightful.
> *[The Call of Cthulhu](https://en.wikisource.org/wiki/The_Call_of_Cthulhu), HP Lovecraft, 1926.*

HDFS uses Kerberos to 

1. Authenticate caller's access to the Namenode and filesystem metadata (directory and file manipulation).
1. Authenticate Datanodes attempting to join the HDFS cluster. This prevents malicious code
 from claiming to be part of HDFS and having blocks passed to it.
1. Authenticate the Namenode with the Datanodes (prevents malicious code claiming to be
the Namenode and granting access to data or just deleting it)
1. Grant callers read and write access to data within HDFS.

Kerberos is used to set up the initial trust between a client and the NN, by way of
Hadoop tokens. A client with an Authentication Token can request a Delegation Token,
which it can then pass to other services or YARN applications, so giving them time-bound
access to HDFS with the rights of that user.

The namenode also issues "Block Tokens" which are needed to access HDFS data stored on the
Datanodes: the DNs validate these tokens, rather than requiring clients to authenticate
with the DNs in any way. This avoids any authentication overhead on block requests,
and the need to somehow acquire/share delegation tokens to access specific DNs.

HDFS Block Tokens do not (as of August 2015) contain information about the identity of the caller or
the process which is running. This is somewhat of an inconvenience, as it prevents
the HDFS team from implementing user-specific priority/throttling of HDFS data access
—something which would allow YARN containers to manage the IOPs and bandwith of containers,
and allow multi-tenant Hadoop clusters to prioritise high-SLA applications over lower-priority
code.

## HDFS NameNode


1. NN reads in a keytab and initializes itself from there (i.e. no need to `kinit`; ticket
renewal handed by `UGI`).
1. Generates a *Secret*

Delegation tokens in the NN are persisted to the edit log, the operations `OP_GET_DELEGATION_TOKEN`
`OP_RENEW_DELEGATION_TOKEN` and `OP_CANCEL_DELEGATION_TOKEN` covering the actions. This ensures
that on failover, the tokens are still valid


### Block Keys

A `BlockKey` is the secret used to show that the caller has been granted access to a block
in a DN. 

The NN issues the block key to a client, which then asks a DN for that block, supplying
the key as proof of authorization.

Block Keys are managed in the `BlockTokenSecretManager`, one in the NN
and another in every DN to track the block keys to which it has access. 
It is the DNs which issue block keys as blocks are created; when they heartbeat to the NN
they include the keys.

### Block Tokens

A `BlockToken` is the token issued for access to a block; it includes 

    (userId, (BlockPoolId, BlockId), keyId, expiryDate, access-modes)

The block key itself isn't included, just the key to the referenced block. The access modes declare
what access rights the caller has to the data

    public enum AccessMode {
      READ, WRITE, COPY, REPLACE
    }

It is the NN which has the permissions/ACLs on each file —DNs don't have access to that data.
Thus it is the BlockToken which passes this information to the DN, by way of the client.
Obviously, they need to be tamper-proof.


## DataNodes

DataNodes do not use Hadoop RPC —they transfer data over HTTP. This delivers better performance,
though the (historical) use of Jetty introduced other problems. At scale, obscure race conditions
in Jetty surfaced. Hadoop now uses Netty for its DN block protocol.

### DataNodes and SASL

Pre-2.6, all that could be done to secure the DN was to bring it up on a secure (&lt;1024) port
and so demonstrate that an OS superuser started the process. Hadoop 2.6 supports SASL
authenticated HTTP connections, which works *provided all clients are all running Hadoop 2.6+*

See [Secure DataNode](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html#Secure_DataNode)

## HDFS Bootstrap

1. NN reads in a keytab and initializes itself from there (i.e. no need to `kinit`; ticket
renewal handed by `UGI`).
1. Generates a *Secret*


Delegation tokens in the NN are persisted to the edit log, the operations `OP_GET_DELEGATION_TOKEN`
`OP_RENEW_DELEGATION_TOKEN` and `OP_CANCEL_DELEGATION_TOKEN` covering the actions. This ensures
that on failover, the tokens are still valid



## HDFS Client interaction

1. Client asks NN for access to a path, identifying via Kerberos or delegation token.
1. NN authenticates caller, if access to path is authorized, returns Block Token to the client.
1. Client talks to 1+ DNs with the block, using the Block Token.
1. DN authenticates Block Token using shared-secret with NameNode.
1. if authenticated, DN compares permissions in Block Token with operation requested, then
grants or rejects the request.

The client does not have its identity checked by the DNs. That is done by the NN. This means
that the client can in theory pass a Block Token on to another process for delegated access to a single
block. It has another implication: DNs can't do IO throttling on a per-user basis, as they do
not know the user requesting data.

### WebHDFS

1. In a secure cluster, Web HDFS requires SPNEGO
1. After authenticating with a SPNEGO-negotiated mechanism, webhdfs sends an HTTP redirect,
including the BlockTocken in the redirect

### NN/Web UI

1. If web auth is enabled in a secure cluster, the DN web UI will requires SPNEGO
1. In a secure cluster, if webauth is disabled, kerberos/SPNEGO auth may still be needed
to access the HDFS browser. This is a point of contention: its implicit from the delegation
 to WebHDFS --but a change across Hadoop versions, as before an unauthed user could still browse
 as "dr who". 
 
