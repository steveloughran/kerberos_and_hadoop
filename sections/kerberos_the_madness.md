
# Hadoop and Kerberos: The Madness Beyond the Gate


Authors:

S.A. Loughran


----

# Introduction

When HP Lovecraft wrote his books about forbidden knowledge which would reduce the reader to insanity, of "Elder Gods" to whom all of humanity were a passing inconvenience, most people assumed that he was making up a fantasy world.
In fact he was documenting Kerberos.

What is remarkable is that he did this fifty years before kerberos was developed. This makes him less of an author, 
instead: a prophet.

What he wrote was true: there are some things humanity was not meant to know. Most people are better off living lives of naive innocence, never having to see an error message about SASL or GSS, never fear building up scripts of incantations to `kadmin.local`, incantations which you hope to keep evil and chaos away. To never stare in dismay at the code whose true name must never be spoken, but instead it's initials whispered, "UGI". For those of us who have done all this, our lives are forever ruined. From now on we will cherish any interaction with a secure Hadoop cluster —from a client application to HDFS, or application launch on a YARN cluster, and simply viewing a web page in a locked down web UI —all as a miracle against the odds, against the forces of chaos struggling to destroy order.
And forever more, we shall fear those voices calling out to us in the night, the machines by our bed talking to us, saying things like "we have an urgent support call related to REST clients on a remote kerberos cluster —can you help?" 


| HP Lovecraft                                          | Kerberos                   |
|-------------------------------------------------------|----------------------------|
| Evil things lurking in New England towns and villages | MIT Project Athena         |
| Ancient, evil deities oblivious to humanity           | Kerberos Domain Controller |
| Books whose reading will drive the reader insane      | IETF RFC 4120              |
| Entities which are never spoken of aloud              | UserGroupInformation       |
| People driven insane by their knowledge               | You                        |

This documents contains the notes from previous people who have delved too deep into the mysteries of Apache&trade; Hadoop&reg; and Kerberos, who have read the forbidden source code, maybe who have even contributed to it. If you wish to preserve your innocence, to view the world as a place of happiness: stop now.

## Disclaimer

This document is a collection of notes based on the experience of the author. There are no guarantees that any of the information contained within was correct at the time of writing, let alone the time of reading. The author does not accept any responsibility for actions made on the basis of the information contained herein, be it correct or incorrect.

The reader of this document is likely to leave with some basic realisation that Kerberos, while important, is an uncontrolled force of suffering and devastation. The author does not accept any responsibility for the consequences of such knowledge.

What has been learned cannot be unlearned(*)

(*) Except for Kerberos workarounds you wrote 18 months ago and for which you now field support calls.

----

# Foundational Concepts

What is the problem that Hadoop security is trying to address? Securing Hadoop.

Apache Hadoop is "an OS for data".
A Hadoop cluster can rapidly become the largest stores of data in an organisation.
That data can explicitly include sensitive information: financial, personal, business, and can often implicitly contain data which needs to be sensitive about the privacy of individuals (for example, log data of web accesses alone).
Much of this data is protected by laws of different countries.
This means that access to the data needs to be strictly controlled, and accesses made of that data potentially logged to provide an audit trail of use.

You have to also consider, "why do people have Hadoop clusters?".

It's not just because they have lots of data --its because they want to make use of it.
A data-driven organisation needs to trust that data, or at least be confident of its origins.
Allowing entities to tamper with that data is dangerous.

For the protection of data, then, read and write access to data stored directly in the HDFS filesystem needs to be protected.
Applications which work with their data in HDFS also need to have their accesses restricted: Apache HBase and Apache Accumulo store their data in HDFS, Apache Hive submits SQL queries to HDFS-stored data, etc.
All these accesses need to be secured; applications like HBase and Accumulo granted restricted access to their data, and themselves securing and authenticating communications with their clients.

YARN allows arbitrary applications to be deployed within a Hadoop cluster.
This needs to be done without granting open access to the entire cluster from those user-launched applications, while isolating different users' work.
A YARN application started by user Alice should not be able to directly manipulate an application launched by user "Bob", even if they are running on the same host.
This means that not only do they need to run as different users on the same host (or in some isolated virtual/container), the applications written by Alice and Bob themselves need to be secure.
In particular, any web UI or IPC service they instantiate needs to have its access restricted to trusted users. here Alice and Bob

## Authentication

The authentication problem: who is a caller identifying themselves as —and can you verify
that they really are this person.

In an unsecure cluster, all callers to HDFS, YARN and other services are trusted to be
who they say they are. In a secure cluster, services need to authenticate callers.
That means some information must be passed with remote IPC/REST calls to declare
a caller's identity and authenticate that identity

## Authorization

Does an (authenticated) user have the permissions to perform the desired request?

This isn't handled by Keberos: this is Hadoop-side, and is generally done
in various ways across systems. HDFS has file and directory permissions, with the
user+group model now extended to ACLs. YARN allows job queues to be restricted
to different users and groups, so restricting the memory & CPU limits of those
users. When cluster node labels are used to differentiate parts of the cluster (e.g. servers with
more RAM, GPUs or other features), then the queues can be used to restrict access
to specific sets of nodes.

Similarly, HBase and Accumulo have their users and permissions, while Hive uses the
permissions of the source files as its primary access control mechanism.

These various mechanisms are all a bit disjoint, hence the emergence of tools
to work across the entire stack for a unified view, Apache Ranger being one example.


## Encryption

Can data be intercepted on disk or over the wire?


### Encrytion of Persistent Data.

HDFS now supports *at rest encryption*; the data is encrypted while stored on disk.

Before rushing to encrypt all the data, consider that it isn't a magic solution to
security: the authentication and authorisation comes first. Encryption adds a new problem,
secure key management, as well as the inevitable performance overhead. It also complicates
some aspects of HDFS use.

Data stored in HDFS by applications is implicitly encrypted. However, applications like 
Hive have had to be reworked to ensure 
that when making queries across encrypted datasets, temporary data files are also stored
in the same encryption zone, to stop the intermediate data being stored unencrypted.
And of course, analytics code running in the servers may also intentionally or unintentionally
persist the sensitive data in an unencrypted form: the local filesystem, OS swap space
and even OS hibernate-time memory snapshots need to be managed.

Before rushing to enable persistent data encryption, then, you need to consider: what is the
goal here? 

What at-REST encryption does deliver is better guarantees that data stored in hard disks
is not recoverable —at least on the HDFS side. However, as OS-level data can persist,
(strongly) wiping HDDs prior to disposal is still going to be necessary to guarantee
destruction of the data.

## Auditing and Governance

Authenticated and Authorized users should not just be able to perform actions
or read and write data —this should all be logged in *Audit Logs* so that
if there is ever a need to see which files a user accessed, or what individual
made specific requests of a service —that information is available. Audit logs
should be 

1. Separate log categories from normal processing logs, so log configurations
can store them in separate locations, with different persistence policies.

1. Machine Parseable. This allows the audit logs themselves to be analyzed. This
does not just have to be for security reasons; Spotify have disclosed that they
run analysis over their HDFS audit logs to identify which files are most popular (and
hence should have their replication factor increased), and which do not get
used more then 7 days after their creation —and hence can be automatically deleted
as part of a workflow.

