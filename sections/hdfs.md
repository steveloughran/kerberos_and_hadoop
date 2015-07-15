# HDFS and Kerberos

> It seemed to be a sort of monster, or symbol representing a monster, of a form which only a diseased fancy could conceive. If I say that my somewhat extravagant imagination yielded simultaneous pictures of an octopus, a dragon, and a human caricature, I shall not be unfaithful to the spirit of the thing. A pulpy, tentacled head surmounted a grotesque and scaly body with rudimentary wings; but it was the general outline of the whole which made it most shockingly frightful.
> *[The Call of Cthulhu](https://en.wikisource.org/wiki/The_Call_of_Cthulhu), HP Lovecraft, 1926.*


(based on work by Kevin Minder)


## HDFS Namenode

1. Namenode

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
