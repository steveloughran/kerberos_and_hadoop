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

## Server not found in Kerberos database (7) 

* DNS is a mess and your machine does not know its own name.
* Your machine has a hostname, but the service principal is a `/_HOST` wildcard and the hostname
is not one there's an entry in the keytab for.

## No valid credentials provided (Mechanism level: Illegal key size)]

Your JVM doesn't have the extended cryptography package and can't talk to the KDC.
Switch to openjdk or go to your JVM supplier (Oracle, IBM) and download the JCE extension package, and install it in the hosts where you want Kerberos to work.

## No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt

This may appear in a stack trace starting with something like:

	javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]

Possible causes:

1. You aren't logged in via `kinit`.
1. You have logged in with `kinit`, but the tickets you were issued with have expired.
1. You did specify a keytab but it isn't there or is somehow otherwise invalid
1. You don't have the Java Cryptography Extensions installed.

## Clock skew too great

	GSSException: No valid credentials provided (Mechanism level: Attempt to obtain new INITIATE credentials failed! (null)) . . . Caused by: javax.security.auth.login.LoginException: Clock skew too great

    GSSException: No valid credentials provided (Mechanism level: Clock skew too great (37) - PROCESS_TGS

    kinit: krb5_get_init_creds: time skew (343) larger than max (300)

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


## Principal not found

The hostname is wrong (or there is >1 hostname listed with different IP addrs) and so a principal
of the form `user/_HOST@REALM` is coming back with the wrong host, and the KDC doesn't find it.

See the comments above about DNS for some more possibilities.

## During SPNEGO Auth: Defective token detected 

    GSSException: Defective token detected (Mechanism level: GSSHeader did not find the right tag)
    
The token supplied by the client is not accepted by the server.

This apparently surfaces in [Java 8 after 8u40](http://sourceforge.net/p/spnego/discussion/1003769/thread/700b6941/#cb84);
if Kerberos server doesn't support the first authentication mechanism which the client
offers, then the client fails. Workaround: don't use those versions of Java.

# Hadoop Web/REST APIs

## AuthenticationToken ignored

This has been seen in the HTTP logs of Hadoop REST/Web UIs:

    WARN org.apache.hadoop.security.authentication.server.AuthenticationFilter: AuthenticationToken ignored: org.apache.hadoop.security.authentication.util.SignerException: Invalid signature

This means that the caller did not have the credentials to talk to a Kerberos-secured channel.

1. The caller may not be logged in.
2. The caller may have been logged in, but its kerberos token has expired, so its authentication headers are not considered valid any more.
3. The time limit of a negotiated token for the HTTP connection has expired. Here the calling app is expected to recognise this, discard its old token and renegotiate a new one. If the calling app is a YARN hosted service, then something should have been refreshing the tokens for you.

