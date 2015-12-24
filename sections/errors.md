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

# Error Messages to Fear

> The oldest and strongest emotion of mankind is fear, and the oldest and strongest kind of fear is fear of the unknown.

> *[Supernatural Horror in Literature](https://en.wikisource.org/wiki/Supernatural_Horror_in_Literature), HP Lovecraft, 1927.*


Security error messages appear to take pride in providing limited information. In particular,
they are usually some generic `IOException` wrapping a generic security exception. There is some
text in the message, but it is often `Failure unspecified at GSS-API level`, which means
"something went wrong".

Generally a stack trace with UGI in it is a security problem, *though it can be a network problem
surfacing in the security code*.

The underlying causes of problems are usually the standard ones of distributed systems: networking
and configuration.


In [HADOOP-12426](https://issues.apache.org/jira/browse/HADOOP-12426) I've proposed a CLI entry point
for health checking this. Volunteers to implement welcome.


# OS/JVM Layer; GSS library

Some of these are covered in Oracle's Troubleshooting Kerberos docs.
This section just highlights some of the common causes, other causes that Oracle don't mention —and messages they haven't covered.

## Server not found in Kerberos database (7) / service ticket not found in the subject

* DNS is a mess and your machine does not know its own name.
* Your machine has a hostname, but the service principal is a `/_HOST` wildcard and the hostname
is not one there's an entry in the keytab for.

## No valid credentials provided (Mechanism level: Illegal key size)]

Your JVM doesn't have the extended cryptography package and can't talk to the KDC.
Switch to openjdk or go to your JVM supplier (Oracle, IBM) and download the JCE extension package, and install it in the hosts where you want Kerberos to work.

## No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt

This may appear in a stack trace starting with something like:

```
javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
```

It's very common, and essentially means "you weren't authenticated"

Possible causes:

1. You aren't logged in via `kinit`.
1. You have logged in with `kinit`, but the tickets you were issued with have expired.
1. Your process was issued with a ticket, which has now expired.
1. You did specify a keytab but it isn't there or is somehow otherwise invalid
1. You don't have the Java Cryptography Extensions installed.


## failure to login using ticket cache file

You aren't logged via `kinit`, the application isn't configured to use a keytab. So: no ticket,
no authentication, no access to cluster services. 

you can use `klist -v` to show your current ticket cache 

fix: log in with `kinit`

## Clock skew too great

```
GSSException: No valid credentials provided (Mechanism level: Attempt to obtain new INITIATE credentials failed! (null)) . . . Caused by: javax.security.auth.login.LoginException: Clock skew too great

GSSException: No valid credentials provided (Mechanism level: Clock skew too great (37) - PROCESS_TGS

kinit: krb5_get_init_creds: time skew (343) larger than max (300)
```

This comes from the clocks on the machines being too far out of sync. 

This can surface if you are doing Hadoop work on some VMs and have been suspending and resuming them;
they've lost track of when they are. Reboot them.

If it's a physical cluster, make sure that your NTP daemons are pointing at the same NTP server, one that is actually reachable from the Hadoop cluster. And that the timezone settings of all the hosts are consistent.



## KDC has no support for encryption type

This crops up on the MiniKDC if you are trying to be clever about encryption types. It doesn't support many.

## Failure unspecified at GSS-API level (Mechanism level: Checksum failed)

1. The password is wrong. A `kinit` command doesn't send the password to the KDC —it sends some hashed things
to prove to the KDC that the caller has the password. If the password is wrong, so is the hash, hence
an error about checksums.
1. Kerberos is very strict about hostnames and DNS; this can somehow trigger the problem.
[http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization](http://stackoverflow.com/questions/12229658/java-spnego-unwanted-spn-canonicalization); 
1. Java 8 behaves differently from Java 6 and 7 here which can cause problems
[(HADOOP-11628](https://issues.apache.org/jira/browse/HADOOP-11628).

## `GSSException: No valid credentials provided (Mechanism level: Fail to create credential. (63) - No service creds)`


Rarely seen. Switching kerberos to use TCP rather than UDP makes it go away

In `krb5.conf`:

```
[libdefaults]
  udp_preference_limit = 1
```

##  `GSSException: No valid credentials provided (Mechanism level: Connection reset)'

We've seen this triggered in Hadoop tests after the MiniKDC through an exception; its thread
exited and hence the Kerberos client got a connection error.

When you see this assume network connectivity problems, or something up at the KDC itself.

## Principal not found

The hostname is wrong (or there is >1 hostname listed with different IP addrs) and so a principal
of the form `user/_HOST@REALM` is coming back with the wrong host, and the KDC doesn't find it.

See the comments above about DNS for some more possibilities.

## During SPNEGO Auth: Defective token detected 

```
GSSException: Defective token detected (Mechanism level: GSSHeader did not find the right tag)
```

The token supplied by the client is not accepted by the server.

This apparently surfaces in [Java 8 version 8u40](http://sourceforge.net/p/spnego/discussion/1003769/thread/700b6941/#cb84);
if Kerberos server doesn't support the first authentication mechanism which the client
offers, then the client fails. Workaround: don't use those versions of Java.

This is [now acknowledged by Oracle](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8080129) and
has been fixed in 8u60.


## Specified version of key is not available (44)

```
Client failed to SASL authenticate: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: Failure unspecified at GSS-API level (Mechanism level: Specified version of key is not available (44))]
```

The meaning of this message —or how to fix it— is a mystery to all.

There is [some tentative coverage in Stack Overflow](http://stackoverflow.com/questions/24511812/krbexception-specified-version-of-key-is-not-available-44)

One possibility is that the keys in your keytab have expired. Did you know that can happen? It does.
One day your cluster works happily. The next your client requests are failing, with this message
surfacing in the logs.

```
 klist -kt zk.service.keytab 
Keytab name: FILE:zk.service.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
   5 12/16/14 11:46:05 zookeeper/devix.cotham.uk@COTHAM
```

One thing to see there is the version number in the KVNO table.

Oracle describe the JRE's handling of version numbers [in their bug database](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6984764).

From an account logged in to the system, you can look at the client's version number

```
$ kvno zookeeper/devix@COTHAM
zookeeper/devix@COTHAM: kvno = 1
```

*Recommended strategy*

Rebuild your keytabs.

1. Take a copy of your current keytab dir, for easy reverting.
1. Use `ktlist -kt` to list the entries in each keytab.
1. Use `ls -al` to record their user + group values + permissions.
1. In `kadmin.local`, re-export every key to the keytabs which needed it with `xst -norandkey`
1. Make sure the file owners and permissions are as before.
1. Restart everything.


## javax.security.auth.login.LoginException: No password provided

When this surfaces in a server log, it means the server couldn't log in as the user. That is,
there isn't an entry in the supplied keytab for that user and the system (obviously) doesn't
want to fall back to user-prompted password entry.
 
Some of the possible causes

* The wrong keytab was specified.
* There isn't an entry in the keytab for the user.
* The hostname of the machine doesn't match that of a user in the keytab, so a match of `service/host`
fails.

Ideally, services list the keytab and username at fault here. In a less than ideal world —that is
the one we live in— things are less helpful

Here, for example, is a Zookeeper trace, saying it is the user `null` that is at fault.

```
2015-12-15 17:16:23,517 - WARN  [main:SaslServerCallbackHandler@105] - No password found for user: null
2015-12-15 17:16:23,536 - ERROR [main:ZooKeeperServerMain@63] - Unexpected exception, exiting abnormally
java.io.IOException: Could not configure server because SASL configuration did not allow the  ZooKeeper server to authenticate itself properly: javax.security.auth.login.LoginException: No password provided
        at org.apache.zookeeper.server.ServerCnxnFactory.configureSaslLogin(ServerCnxnFactory.java:207)
        at org.apache.zookeeper.server.NIOServerCnxnFactory.configure(NIOServerCnxnFactory.java:87)
        at org.apache.zookeeper.server.ZooKeeperServerMain.runFromConfig(ZooKeeperServerMain.java:111)
        at org.apache.zookeeper.server.ZooKeeperServerMain.initializeAndRun(ZooKeeperServerMain.java:86)
        at org.apache.zookeeper.server.ZooKeeperServerMain.main(ZooKeeperServerMain.java:52)
        at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:116)
        at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:78)

```
### `kinit: Client not found in Kerberos database while getting initial credentials`

This is fun: it means that the user is not known.

Possible causes

1. The user isn't in the database.
1. You are trying to connect to a different KDC than the one you thought you were using.
1. You aren't who you thought you were.

# Hadoop Web/REST APIs

## AuthenticationToken ignored

This has been seen in the HTTP logs of Hadoop REST/Web UIs:

```
WARN org.apache.hadoop.security.authentication.server.AuthenticationFilter: AuthenticationToken ignored: org.apache.hadoop.security.authentication.util.SignerException: Invalid signature
```

This means that the caller did not have the credentials to talk to a Kerberos-secured channel.

1. The caller may not be logged in.
2. The caller may have been logged in, but its kerberos token has expired, so its authentication headers are not considered valid any more.
3. The time limit of a negotiated token for the HTTP connection has expired. Here the calling app is expected to recognise this, discard its old token and renegotiate a new one. If the calling app is a YARN hosted service, then something should have been refreshing the tokens for you.

