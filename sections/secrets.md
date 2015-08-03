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
  

# Low-Level Secrets



> Among the agonies of these after days is that chief of torments — inarticulateness. What I learned and saw in those hours of impious exploration can never be told — for want of symbols or suggestions in any language.

> *[The Shunned House](https://en.wikipedia.org/wiki/The_Shunned_House), HP Lovecraft, 1924.*

## `krb5.conf` and system property `java.security.krb5.conf`


You can do two things when setting up the JVM binding to the `krb5.conf` kerberos
binding file.


*1. Change the realm with System Property `java.security.krb5.realm`*

This system property sets the realm for the kerberos binding. This allows you to use a different one from the default in the krb5.conf file. 


Examples

    -Djava.security.krb5.realm=PRODUCTION

    System.setProperty("java.security.krb5.realm", "DEVELOPMENT");

The JVM property MUST be set before UGI is initialized.


*2. Switch to an alternate `krb5.conf` file.

The JVM kerberos operations are configured via the `krb5.conf` file specified in the JVM option
`java.security.krb5.conf` which can be done on the JVM command line, or inside the JVM

		System.setProperty("java.security.krb5.conf", krbfilepath);

The JVM property MUST be set before UGI is initialized.

Notes

* use double backslash to escape paths on Windows platforms, e.g. `C:\\keys\\key1`, or `\\\\server4\\shared\\tokens`
* Different JVMs (e.g. IBM JVM) want different fields in their `krb5.conf` file. How can you tell? Kerberos will fail with a message


## JVM Kerberos Library logging

You can turn Kerberos low-level logging on

		-Dsun.security.krb5.debug=true

This doesn't come out via Log4J, or `java.util logging;` it just comes out on the console. Which is somewhat inconvenient —but bear in mind they are logging at a very low level part of the system. And it does at least log.
If you find yourself down at this level you are in trouble. Bear that in mind.


### JVM SPNEGO Logging

If you want to debug what is happening in SPNEGO, another system property lets you enable this:

    -Dsun.security.spnego.debug=true

## KRB5CCNAME

The environment variable [`KRB5CCNAME`](http://web.mit.edu/kerberos/krb5-1.4/krb5-1.4/doc/klist.html)
As the docs say:

If the KRB5CCNAME environment variable is set, its value is used to name the default ticket cache.

## IP addresses vs. Hostnames

Kerberos principals are traditionally defined with hostnames of the form `hbase@worker3/EXAMPLE.COM`, not `hbase/10.10.15.1/EXAMPLE.COM`

The issue of whether Hadoop should support IP addresses has been raised [HADOOP-9019](https://issues.apache.org/jira/browse/HADOOP-9019) & [HADOOP-7510](https://issues.apache.org/jira/browse/HADOOP-7510)
Current consensus is no: you need DNS set up, or at least a consistent and valid /etc/hosts file on every node in the cluster.

Warning: Windows does not reverse-DNS 127.0.0.1 to localhost or the local machine name; this can cause problems with MiniKDC tests in Windows, where adding a `user/127.0.0.1@REALM` principal will be needed [example](https://github.com/apache/hadoop/blob/trunk/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-registry/src/test/java/org/apache/hadoop/registry/secure/AbstractSecureRegistryTest.java#L209).

## Kerberos's defences against replay attacks

from the javadocs of `org.apache.hadoop.ipc.Client.handleSaslConnectionFailure()`:

    /**
     * If multiple clients with the same principal try to connect to the same
     * server at the same time, the server assumes a replay attack is in
     * progress. This is a feature of kerberos. In order to work around this,
     * what is done is that the client backs off randomly and tries to initiate
     * the connection again.
     */

That's a good defence on the surface, "multiple connections from same principal == attack", which
doesn't scale to Hadoop clusters. Hence the sleep

 