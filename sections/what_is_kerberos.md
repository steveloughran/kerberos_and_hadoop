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
—a right which the principal can grant for a limited amount of time.

In Hadoop, feature #2 is key: a user or a process may *delegate* the authority to another
process, which can then talk to the desired service with the delegated authority. These
delegation rights are both limited in scope --- the principal delegates authority on a
service-by-service basis --- and in time. The latter is for security reasons ---it guarantees
that if the secret used to act as a delegate, the *token*, is stolen, there is
only a finite time for which it can be used.

How does it work? That is beyond the scope of this book and its author.

It is covered in detail in [Coluris01], S7.6.2.
However, anyone attempting to read this will generally come out with at least a light headache
and no better informed.
 

For an approximate summary of the concepts

## Kerberos Domain Controller, *the KDC*

The KDC is the gate, it is is the key and guardian of the gate, it is the gateway to the
madness that is Kerberos.

Every Kerberos Realm needs at least one. There's one for Linux and Active Directory
can act as a federated KDC infrastructure. Hadoop cluster management tools often
aid in setting up a KDC for a Hadoop cluster. 
There's even a minature one, the `MiniKDC` in the Hadoop source for [testing](testing.html).

KDCs are managed by operations teams. If a developer finds themselves maintaining a KDC outside
of a test environment, they are in trouble and probably out of their depth.

## Kerberos Principal

A principal is an identity in the system; a person or a thing like the hadoop namenode
which has been given an identity.

In Hadoop, a different principal is usually created for each service and machine in the cluster,
such as `hdfs/node1`, `hdfs/node2`, ... etc. These principals would then be used for
all HDFS daemons running on node1, node2, etc.

It's possible to shortcut this and skip the machine specific principal, downgrading
to one per service, such as `hdfs`, `yarn`, `hbase` —or even one for all hadoop applications,
such as `hadoop`. This can be done on a small cluster, but doesn't scale well, or makes
working out WTF is going on difficult.

In particular, the bits of Kerberos which handle logins, the Kerberos Domain Controllers,
treat repeated attempts to log in as the same principal within a short period of time
as some attack on the system, such as a replay or key guessing attack. The requests
are all automatically rejected (presumably without any validation, so as to reduce
CPU load on the server). Even a small Hadoop cluster could generate enough authentication
requests on a cluster restart for this to happen —hence a different principal for every
service on every node.

How do the Hadoop services know which principal to identify themselves at? Generally,
though Hadoop configuration files. They also determine the hostname, and use this to
decide which of the possible principals in their keytab (see below) to identify themselves
at. For this to work, machines have to know who they are.

Specifically
1. They have to have a name
1. That name has to be in their host table or DNS
1. It has to match the IP address of the host

#### what if there is more than one IP address on the host?

Generally Hadoop services
are single-IP address, which is a limitation that is likely to be addressed at some time.
So who knows. Actually, it comes from `org.apache.hadoop.net.NetUtils.getHostname()`, which
invokes `InetAddress.getLocalHost()` and relies on this to return a hostname. 
It's because of this need to know hostnames for principals, that requests for
Hadoop services to use IP Addresses over hostnames, such as [MAPREDUCE-6463](https://issues.apache.org/jira/browse/MAPREDUCE-6463)
are declined. It also means that if a machine does not know its own hostname, things
do not work [HADOOP-3426](https://issues.apache.org/jira/browse/HADOOP-3426),
[HADOOP-10011](https://issues.apache.org/jira/browse/HADOOP-10011).

## Kerberos Realm

A Kerberos *Realm* is the security equivalent of a subnet: all principals live in a realm.
It is conventional, though not mandatory, to use capital letters and a single name, rather
than a dotted network address. Examples: `ENTERPRISE`, `HCLUSTER`

Kerberos allows different realms to have some form of trust of others. This would allow
a Hadoop cluster with its own KDC and realm to trust the `ENTERPRISE` realm, but for the
enterprise realm to not trust the HCLUSTER realm, and hence all its principals. This would
prevent a principal `hdfs/node1@HCLUSTER` from having access to the `ENTERPRISE` systems.
While this is a bit tricky to set up, it means that keytabs created for the Hadoop cluster
(see below) are only a security risk for the Hadoop cluster and all data kept in/processed
by it, rather than the entire organisation.

## Kerberos login, `kinit`

The command line program `kinit` is how a user authenticates with a KDC on a unix system;
it uses the information stored in `/etc/krb`

Alongside `kinit`, comes `kdestroy`, to destroy credentials/log out, and `klist` to list the current
status. The `kdestroy` command is invaluable if you want to verify that any program you start
on the command line really is reading in and using keytab.

Here's what a full `klist -v` listing looks like

```
$ klist -v
Credentials cache: API:489E6666-45D0-4F04-9A1D-FCD5D48EEA07
        Principal: stevel@COTHAM
    Cache version: 0

Server: krbtgt/COTHAM@COTHAM
Client: stevel@COTHAM
Ticket etype: aes256-cts-hmac-sha1-96, kvno 1
Ticket length: 326
Auth time:  Sep  2 11:52:02 2015
End time:   Sep  3 11:52:01 2015
Renew till: Sep  2 11:52:02 2015
Ticket flags: enc-pa-rep, initial, renewable, forwardable
Addresses: addressless

Server: HTTP/devix.cotham.uk@COTHAM
Client: stevel@COTHAM
Ticket etype: aes256-cts-hmac-sha1-96, kvno 25
Ticket length: 333
Auth time:  Sep  2 11:52:02 2015
Start time: Sep  2 12:20:00 2015
End time:   Sep  3 11:52:01 2015
Ticket flags: enc-pa-rep, transited-policy-checked, forwardable
Addresses: addressless

```

A shorter summary comes from the basic `klist`

```
$ klist
Credentials cache: API:489E6666-45D0-4F04-9A1D-FCD5D48EEA07
        Principal: stevel@COTHAM

  Issued                Expires               Principal
Sep  2 11:52:02 2015  Sep  3 11:52:01 2015  krbtgt/COTHAM@COTHAM
Sep  2 12:20:00 2015  Sep  3 11:52:01 2015  HTTP/devix.cotham.uk@COTHAM
```

This shows that
1. The user is logged in as `stevel@COTHAM`
1. They have a ticket to work with the ticket granting service, `krbtgt/COTHAM@COTHAM`.
1. They have a ticket to authenticate with the principal
`HTTP/devix.cotham.uk@COTHAM`. This is used by some HTTP services running on the host
(`devix.cotham.uk`), specifically the Hadoop Namenode and Resource Manager web pages.
These have both been configured to require Kerberos authentication via [SPNEGO](web_and_rest.html),
and to use `HTTP` as the user. The full principal `HTTP/devix.cotham.uk` is determined
from the host running the service.

## Keytab

A (binary) file containing the secrets needed to log in as a principal

1. It contains all the information to log in as a principal, so is a sensitive file.
1. It can hold many principals, so one can be created for, say, hdfs,
  which contails all its principals, `hdfs/node1@HCLUSTER`, `hdfs/node2@HCLUSTER`, ...etc.
  Thus only one keytab per service is needed.
1. It is created by the KDC administrators, who must then securely propagate that file
to where it can be used.

Keytabs are the only way in which programs can directly authenticate themselves with Kerberos,
(though they can indirectly do this with credentials passed to them). This means
that for any long-lived process, a keytab is needed.

Operations teams are generally very reluctant to provide keytabs. They will need to
create them for all long-lived services which run in the cluster. For services
such as HDFS and YARN this is generally done at cluster setup time. YARN services
have to deal with this problem whenever a user wants to run a long lived *YARN service*
within the cluster: the technical one of keytab management and the organisational one
of getting the keytab in the first place.


To look at and work with keytabs, the `ktutil` command line program is the tool of choice.

## Tickets

Kerberos is built around the notion of *tickets*.

A ticket is something which can be passed to a server to identify that the caller
and to provide a secret key that can be used between the client an the server 
—for the duration of the ticket's lifetime. It is all that a server needs to
authenticate a client: there's no need for the server to talk to the KDC.

What's important is that tickets can be passed on: an authenticated principal
can obtain a ticket to a service, and pass that on to another process in the distributed
system. The recipient can then issue requests on behalf of the original principal,
using that ticket. That recipient only has the permissions granted to the ticket
(it does not have any other permissions of the principal, unless those tickets are
also provided), and those permissions are only valid for as long as the ticket
is valid.

The limited lifetime of tickets ensures that even if a ticket is captured by a malicious
attacker, they can only make use of the credential for the lifetime of the ticket.
The ops team doesn't need to worry about lost/stolen tickets, to have a process for
revoking them, as they expire within a short time period, usually a couple of days.

This notion of tickets starts right at the bottom of Kerberos. When a principal
authenticates with the KDC, it doesn't get any special authentication secrets
—it gets a ticket to the *Ticket Granting Service*. This ticket can then be used
to get tickets to other services —and, like any other ticket, can be forwarded.
Equally importantly, the ticket will expire —forcing the principal to re-authenticate
via the command line or a keytab.

## Kerberos Authentication Service

This is network-accessible service which runs in the KDC, and which is used
to authenticate callers. The protocol to authenticate callers is one of those
low level details found in text books. What is important to know is that

1. The KDC contains 'a secret' shared with the principal. There is no public/private
key system here, just a shared secret.
1. When a client authenticates on the command line, the password is (somehow) used
to generate some data which is passed to the authentication service to show that
at least during the authentication process, the client had the password for the principal
they were trying to authenticate as. (i.e., part of the process includes a challenge issued by the
KDC, a challenged hashed by the password to show that's in the callers' possession).
1. When a client authenticates via the keytab, a similar challenge-reponse operation
takes place to allow the client to show they have the principal's (secret) data in that keytab.
1. When the KDC's secret key for a principal is changed, all existing keytabs stop working.

## Ticket Granting Service, *TGS*

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

## Kerberos User Login

A user logs in with the Kerberos Authentication Service


## Examples of Kerberos

To put things into the context of Hadoop, here are some examples of how it could be used.


### Example: User listing an HDFS directory

A user wishes to submit some work to a Hadoop cluster, a new YARN application.

First, they must be logged in to the Kerberos infrastructure,

1. On unix, this is done by running `kinit`
1. The `kinit` program asks the user for their password.
1. This is used to authenticate the user with the *Authentication Service* of the
KDC configured in `/etc/krb5.conf`.
1. The Kerberos *Authentication Service* authenticates the user and issues a TGT ticket,
which is stored in the client's *Credentials Cache*. A call to `klist` can be used to verify this.

Then, they must run a hadoop command

    hadoop fs -ls /

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
implicitly, authenticate the NameNode to the caller.
1. The Namenode can use the Kerberos information to determine the identity of the (authenticated)
caller.
1. It can then look at the permissions of the user as recorded in the HDFS directory and file metadata
and determine if they have the rights to perform the requested action.
1. If they do, the action is performed and the results returned to the caller.

(Note there's some glossing over of details here, specifically how the client to Namenode
authentication takes place, how they stay authenticated, how a users principal gets mapped to user name and
how its group membership is ascertained for authorization purposes.)


If a second request is made against the Namenode in the same Java process, there is no
need to ask the TGT for a new ticket —not until the previous one expires. Instead cached
authentication data is reused. This avoids involving the KDC in any further interactions with the
Namenode.

In Hadoop —as we will see— things go one step further, with Delegation Tokens. For now: ignore them.

This example shows Kerberos at work, and the Hadoop IPC integration.

As described, this follows the original Kerberos architecture, one principal per user, tickets
between users and services. Hadoop/Kerberos integration has to jump one step further to
address the scale problem, to avoid overloading the KDC with requests, to avoid
problems such as having to have the client ask the TGT for a ticket to talk to individual
Datanodes when reading or writing a file across the HDFS filesystem, or even handle the problem
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
enough so that you should really test with a cluster using AD as the Kerberos infrastructure, alongside
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
1. Because a stolen ticket can be used directly against a service, there's no log of its use
in the KDC. Every application needs to have its own audit log of actions performed by
a user, so that the history of actions by a client authenticated with a stolen ticket
can be traced.
1. It's an authentication service: it verifies callers and allows callers to pass that authentication
information on. It doesn't deal with permissions *at all*.


There's some coverage of other issues in
[Kerberos in the Crosshairs: Golden Tickets, Silver Tickets, MITM, and More](https://digital-forensics.sans.org/blog/2014/11/24/kerberos-in-the-crosshairs-golden-tickets-silver-tickets-mitm-more)

## Hadoop/Kerberos Integration Issues

Hadoop specific issues are:

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
