# HDFS and Kerberos

(based on work by Kevin Minder)


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
