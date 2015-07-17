
# What is Kerberos?

> Yog-Sothoth knows the gate.
> Yog-Sothoth is the gate.
> Yog-Sothoth is the key and guardian of the gate.
> Past, present, future, all are one in Yog-Sothoth.
> He knows where the Old Ones broke through of old, and where They shall break through again.
> He knows where They have trod earth's fields, and where They still tread them, and why no one can behold Them as They tread.

>  *["The Dunwich Horror"](https://en.wikisource.org/wiki/The_Dunwich_Horror), HP Lovecraft, 1928*


Kerberos is a system for authenticating access to distributed services: 

1. that callers to a service represent a `principal` in the system, or
1. That a caller to a service has been granted the right to act on behalf of a principal 
-a right which the principal can grant for a limited amount of time.

In Hadoop, feature #2 is key: a user or a process may *delegate* the authority to another
process, which can then talk to the desired service with the delegated authority. These
delegation rights are both limited in scope --- the principal delegates authority on a
service-by-service basis --- and in time. The latter is for security reasons ---it guarantees
that if the secret used to act as a delegate, the *token*, is stolen, there is
only a finite time for which it can be used.

How does it work? That is beyond the scope of this book and its author.

It is covered in detail in Colouris et al, 2001, *Distributed System Concepts and Design*, S7.6.2.
However, anyone attempting to read this will come out no better informed.

For an approximate summary of the concepts

1. A *Kerberos Domain Controller*, *KDC* exists on the network to manage Kerberos security
1. It contains an *Authentication Service*, which authenticates remote principals, and
 a *Ticket Granting Service*, *TGS*, which grants access to specific services.
1. The Authentication Service can authenticate via a password-based login, or though the principal having a stored copy of a shared secret, a *key*.
1. The TGS can issue *tickets*, secrets which declare that a caller has duration-limited access
to a requested service, with the rights of the authenticated principal.
1. An authenticated principal can request tickets to services, which they can then use to authenticate
directly with those services, and interact with them until the ticket expires.
1. A principal can also forward a ticket to any other process/service within the distributed system,
to *delegate* rights.
1. This delegate can use the ticket to access the service, with the identity of the principal, for
the lifespan of that ticket.

Note that Hadoop goes beyond this with the notion of *delegation tokens*, secrets which are similar
to *tickets*, but which can be issued and renewed directly by Hadoop services. That will
be covered in a later chapter.


## Examples of Kerberos

To put things into the context of Hadoop, here are some examples of how it could be used.


### User listing an HDFS directory

A user wishes to submit some work to a Hadoop cluster, a new YARN application.

First, they must be logged in to the Kerberos infrastructure,

1. On unix, this is done by running `kinit`
1. The `kinit` program asks the user for their password.
1. This is used to authenticate the user with the *Authentication Service* of the
KDC configured in `/etc/krb5.conf`.
1. The Kerberos *Authentication Service* authenticates the user and issues a TGT ticket,
which is stored in the client's *Credentials Cache*. A call to `klist` can be used to verify this.

Then, they must run a hadoop command

    hadoop fs --ls /

1. The HDFS client code attempts to talk to the HDFS Namenode via the
`org.apache.hadoop.hdfs.protocol.ClientProtocol` IPC protocol
1. It checks to see if If security is enabled (via `UserGroupInformation.isSecurityEnabled()`)
1. If it is, it looks in metadata assocated with the protocol, metadata which is used
to identify the Kerberos principal, the identity, of the namenode.

            @InterfaceAudience.Private
            @InterfaceStability.Evolving
            @KerberosInfo(serverPrincipal ="dfs.namenode.kerberos.principal")
            @TokenInfo(DelegationTokenSelector.class)
            public interface ClientProtocol {
            ...
            }
1. The Hadoop `Configuration` class instance used to initialise the client is
used to retrieve the value of `"dfs.namenode.kerberos.principal"` —so identifying
the service to which the client must have a valid ticket to talk to.
1. The Hadoop Kerberos code (this is in Java, not the OS), asks the Kerberos *Ticket Granting
Service*, *the TGS*, for a ticket to talk to the Namenode's principal. It does this in a request
authenticated with the *TGT* received during the `kinit` process.
1. This ticket is granted by the TGT, and cached in the memory of the JVM.
1. The Hadoop RPC layer then uses the ticket to authenticate the caller to the Namenode, and
implicitly, authenticate the Namenode to the caller.
1. The Namenode can use the kerberos information to determine the identity of the (authenticated)
caller.
1. It can then look at the permissions of the user as recorded in the HDFS directory and file metadata
and determine if they have the rights to perform the requested action.
1. If they do, the action is performed and the results returned to the caller.

(Note there's some glossing over of details here, specifically how the client to Namenode
authentication takes place, and how they stay authenticated)


If a second request is made against the Namenode in the same Java process, there is no
need to ask the TGT for a new ticket —not until the previous one expires. Instead the cached
authentication data is reused. This avoids involving the KDC in any further interactions with the
Namenode.

This example shows Kerberos at work, and the Hadoop IPC integration.

As described, this follows the original Kerberos architecture, one principal per user, tickets
between users and services. Hadoop/Kerberos integration has to jump one step further to
address the scale problem, to avoid overloading the KDC with requests, to avoid
problems such as having to have the client ask the TGT for a ticket to talk to individual
Datanodes when reading or writing a file across the the HDFS filesystem, or even handle the problem
with a tens of thousands of clients having to refresh their Namenode tickets every few hours.

This is done with a concept called *Hadoop Delegation Tokens*. These will be covered later.

For now, know that the core authentication between principals and services utterly depends
upon the Hadoop infrastructure, with an initial process as describe above.


## Kerberos and Windows Active Directory

A lot of people are blissfully unaware of Kerberos. Such a life is one to treasure. Many of
these people, do, however, log in to an enterprise network by way of Microsoft Active Directory.
"AD" is a Kerberos Controller. [Kerberos Explained](https://msdn.microsoft.com/en-us/library/bb742516.aspx)

If an organisation uses Active Directory to manage users, they are running Kerberos, so
have the basic infrastructure needed to authenticate users and services within a Hadoop cluster.
Users should be able to submit jobs as themselves, interacting with "Kerberized" Hadoop services.

Setting up Hadoop to work with Active Directory is beyond the scope of this book. Please
consult the references in the bibliography, and/or any vendor-specific documentation.

For Developers, it is worth knowing that AD is subtly different from the MIT/Unix Kerberos controller,
enough so that you should really test with a cluster using AD as the Kerbers infrastructure, alongside
the MIT KDC.

## Limitations of Kerberos

Kerberos is considered "the best there is" in terms of securing distributed systems. Its
use of tickets is designed to limit the load on the KDC, as it is only interacted with when
a principal requests a ticket, rather than having to validate every single request.

The ability to delegate tokens to other processes allows transitive authentication as the original
principal. This can be used by core Hadoop services to act on a users behalf, and by processes
launched by the user.

The fact that tickets/tokens are time limited means that if one is stolen, the time for which
unauthorized access is possible is limited to the lifespan of the token.

Finally, the fact that kerberos clients are standard in Windows, Linux and OS/X, and built
into the Java runtime, means that it is possible to use Kerberos widely.

This does not mean it is perfect. Known limitations are

1. The KDC is a Single Point of Failure, unless an HA system is set up (which Active Directory
can do).
1. Excess load can overload the KDC. 
1. The communications channels between services still need to be secure. Kerberos does not
address data encryption. If those channels are not secure, then tickets can be intercepted or
communications forged.
1. Time needs to be roughly consistent across machines, else the time-limited tokens won't work.
1. If time cannot be securely managed across machines (i.e. an insecure time synchronization,
protocol is used), then it is theoretically possible to extend the lifetime of a stolen token.


Hadoop specific issues are

1. While the ticketing process reduces KDC load, an entire
   Hadoop cluster starting up can generate the login requests of a few thousand principals over
   a short period of time. The Hadoop code contains some back-off logic to handle connection and
   authentication failures here.
1. Because granted tokens expire, long-lived YARN services need to have a mechanism for updating
tokens.
1. It's both hard to code for kerberos, and test against it.

Finally, it is *necessary but not sufficient*.

Having a Kerberized application does not guarantee that it is secure: you need to think about
possible weaknesses, ways in which untrusted callers can make use of the service, ways
in which tokens and keytabs may be leaked (that includes log messages!) and defend against them.