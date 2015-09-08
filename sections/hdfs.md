# HDFS and Kerberos

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

## HDFS Namenode

### TODO

1. Namenode reads in a keytab and initializes itself from there (i.e. no need to `kinit`; ticket
renewal handed by `UGI`).
1. In a secure cluster, Web HDFS requires SPNEGO
1. If web auth is enabled in a secure cluster, both the DN web UI will requires SPNEGO
1. In a secure cluster, if webauth is disabled, kerberos/SPNEGO auth may still be needed
to access the HDFS browser. This is a point of contention: its implicit from the delegation
 to WebHDFS --but a change across Hadoop versions, as before an unauthed user could still browse
 as "dr who". 



## DataNodes

DataNodes do not use Hadoop RPC —they transfer data over HTTP. This delivers better performance,
though the (historical) use of Jetty introduced other problems. At scale, obscure race conditions
in Jetty surfaced. Hadoop now uses Netty for its DN block protocol.

Pre-2.6, all that could be done to secure the DN was to bring it up on a secure (&lt;1024) port
and so demonstrate that an OS superuser started the process. Hadoop 2.6 supports SASL
authenticated HTTP connections, which works *provided all clients are all running Hadoop 2.6+*


See [Secure DataNode](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SecureMode.html#Secure_DataNode)

### TODO

## HDFS Client interaction

1. Client asks NN for access to a path, identifying via KST or DT.
1. NN authenticates caller, if access to path is authorized, returns BT to the client.
1. Client talks to 1+ DNs with the block, using the BT.
1. DN authenticates BT using shared-secret with NN.
1. if authenticated, DN compares permissions in BT with operation requested, grants or rejects it.

The client does not have its identity checked by the DNs. That is done by the NN. This means
that the client can in theory pass a BT on to another process for delegated access to a single
file. It has another implication: DNs can't do IO throttling on a per-user basis, as they do
not know the user requesting data.
