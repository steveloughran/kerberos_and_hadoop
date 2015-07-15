
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


As an example, imagine a user deploying a YARN application in a cluster, one which needs
access to the user's data stored in HDFS. The user would be required to be authenticated with
the KDC, and have been granted a *Ticket Granting Ticket*; the ticket needed to work with
the TGS. 

The client-side launcher of the YARN application would be able to talk to HDFS and the YARN
resource manager, because the user was logged in to Kerberos. This would be managed in the Hadoop
RPC layer, requesting tickets to talk to the HDFS NameNode and YARN ResourceManager, if needed.

To give the YARN application the same rights to HDFS, the client-side application must
request a new ticket to talk to HDFS, a key which is then passed to the YARN application in
the `ContainerLaunchContext` within the `ApplicationSubmissionContext` used to define the
application to launch: its required container resources, artifacts to download, "localize",
environment to set up and command to run.

The YARN resource manager finds a location for the Application Master, and requests that
hosts' Node Manager start the container/application. 

The Node Manager uses the "delegated HDFS token" to download the launch-time resources into
a local directory space, then executes the application.

*Somehow*, the HDFS token (and any other supplied tokens) are passed to the application that
has been launched.

The launched application master can use this token to interact with HDFS *as the original user*.

The AM can also pass token(s) on to launched containers, so that they too have access to HDFS.


The Hadoop Name Node does not need to care whether the caller is the user themselves, the Node Manager
localizing the container, the launched application or any launched containers. All it does is verify
that when a caller requests access to the HDFS filesystem metadata or the contents of a file,
it must have a ticket/token which declares that they are the specific user, and that the token
is currently considered valid (based on the expiry time and the clock value of the Name Node)

## Kerberos and Hadoop

As shown above, Hadoop can use Kerberos to authenticate users, and processes running within a
Hadoop cluster acting on behalf of the user. It is also used to authenticate services running
within the Hadoop cluster itself -so that only authenticated HDFS Datanodes can join the HDFS
filesystem, that only trusted Node Managers can heartbeat to the YARN Resource Manager and
receive work.

* The exact means by which all this is done is one of the most complicated pieces of code to span the
entire Hadoop codebase.*
 
Users of Hadoop do not need to worry about the implementation details, and, ideally, nor should
the operations team.

Developers of core Hadoop code, anyone writing a YARN application, and anyone writing code
to interact with a Hadoop cluster and applications running in it *do need to know those details*.

This is what this book attempts to cover.



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