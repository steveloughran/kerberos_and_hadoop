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

# SASL: Simple Authentication and Security Layer

This is an [IETF-managed](http://www.iana.org/assignments/sasl-mechanisms/sasl-mechanisms.xhtml)
specification for securing network channels, with [RFC4422](http://tools.ietf.org/html/rfc4422)
covering the core of it.

SASL is not an authentication mechanism. SASL is a mechanism for applications to set up
an authenticated communications channel by way of a shared authentication mechanism.
SASL covers the protocol for the applications to negotiate as to which authentication
mechanism to use, then to perform whatever challenge/response exchanges are needed for
that authentication to take place. Kerberos is one authentication mechanism, but SASL
supports others, such as x.509 certificates.

Similarly, SASL does not address wire-encryption, or anything related to authorization.
SASL is purely about a client authenticating its actual or delegated identity with a server,
while the client verifies that the server also has the required identity (usually one
declared in a configuration file).

As well as being independent of the authentication mechanism, SASL is independent of the
underlying wire format/communications protocol. The SASL implementation libraries
can be used by applications to secure whatever network protocol they've implemented.

In Hadoop, "SASL" can be taken to mean "authentication negotiated using SASL".
It doesn't define which protocol itself is authenticated —and you don't really need to care.
Furthermore, if you implement your own protocol, if you add SASL-based authentication to it,
you get to use Kerberos, x509, Single-Sign-On, Open Auth (when completed), etc.

For Hadoop RPC, there are currently two protocols for authentication:

* KERBEROS: Kerberos ticket-based authentication
* DIGEST-MD5: MD5 checksum-based authentication; shows caller has a secret which the
  recipient also knows.

Note that there is also the protocol `PLAIN`; SASL-based negotiation to not have any authentication
at all. That doesn't surface in Hadoop —yet— though it does crop up in JIRAs.

## SASL-enabled services

Services which use SASL include

1. Hadoop RPC
1. [Zookeeper](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zookeeper+and+SASL)
1. HDFS 2.6+ DataNode bulk IO protocol (HTTP based)
