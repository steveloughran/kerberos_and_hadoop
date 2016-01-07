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
  
# Testing


## Writing Kerberos Tests with MiniKDC

The Hadoop project has an in-VM Kerberos Controller for tests, MiniKDC, which is packaged as its own JAR for downstream use. The core of this code actually comes from the Apache Directory Services project.

----

## Testing against Kerberized Hadoop clusters

This is not actually the hardest form of testing; getting the MiniKDC working holds that honour.
It does have some pre-requisites.

1. Everyone running the tests has set up a Hadoop cluster/single VM with Kerberos enabled.
2. The software project has a test runner capable of deploying applications into a remote Hadoop cluster/VM and assessing the outcome.

It's primarily the test runner which matters. Without that you cannot do functional tests against any Hadoop cluster.
However, once you have such a test runner, you have a powerful tool: the ability to run tests against real Hadoop clusters, rather than simply minicluster and miniKDC tests which, while better than nothing, are unrealistic.

If this approach is so powerful, why not bypass the minicluster tests altogether?

1. Minicluster tests are easier to run. Build tools can run them; Jenkins can trivially run them as part of test runs.
2. The current state of the cluster affects the outcome of the tests. Its useful not only to have tests tear down properly, but for the setup phase of each test suite to verify that the cluster is in the valid initial state/get it into that state. For YARN applications, this generally means that there are no running applications in the cluster.
3. Setup often includes the overhead of copying files into HDFS. As the secure user.
4. The host launching the tests needs to be setup with kinit/keytabs.
5. Retrieving and interpreting the results is harder. Often it involved manually going to the YARN RM to get through to the logs (assuming that yarn-site is configured to preserve them), and/or collecting other service logs.
6. If you are working with nightly builds of Hadoop, VM setup needs to be automated.
7. Unless you can mandate and verify that all developers run the tests against secure clusters, they may not get run by everyone.
8. The tests can be slow.
9. Fault injection can be harder.

Overall, tests are less deterministic.

In the slider project, different team members have different test clusters, Linux and Windows, Kerberized and non-Kerberized, Java-7 and Java 8. This means that test runs do test a wide set of configurations without requiring every developer to have a VM of every form. The Hortonworks QE team also run these tests against the nightly HDP stack builds, catching regressions in both the HDP stack and in the Slider project.

For fault injection the Slider Application Master has an integral "chaos monkey" which can be configured to start after a defined period of time, then randomly kill worker containers and/or the application master. This is used in conjunction with the functional tests of the deployed applications to verify that they remain resilient to failure. When tests do fail, we are left with the problem of retrieving the logs and identifying problems from them. The QE test runs do collect all the logs from all the services across the test clusters —but this still leaves the problem of trying to correlate events from the logs across the machines.

# Tuning a Hadoop cluster for aggressive token timeouts

## Kinit

You can ask for a limited lifespan of a ticket when logging in on the console
 
    kinit -l 15m

## KDC


Here is an example `/etc/krb5.conf` which limits the lifespan of a ticket
to 1h

```
[libdefaults]

  default_realm = DEVIX
  renew_lifetime = 2h
  forwardable = true

  ticket_lifetime = 1h
  dns_lookup_realm = false
  dns_lookup_kdc = false

[realms]

  DEVIX = {
    kdc = devix
    admin_server = devix
  }
```

The KDC host here, `devix` is a Linux VM. Turning off the DNS lookups avoids
futile attempts to work with DNS/rDNS.

## Hadoop tokens


*TODO: Table of properties for hdfs, yarn, hive, ... listing token timeout properties*

## Enabling Kerberos for different Hadoop components

### Core Hadoop


```xml
<property>
  <name>hadoop.security.authentication</name>
  <value>kerberos</value>
</property>
<property>
  <name>hadoop.security.authorization</name>
  <value>true</value>
</property>
```


### HBase

```xml
<property>
  <name>hbase.security.authentication</name>
  <value>kerberos</value>
</property>
<property>
  <name>hbase.security.authorization</name>
  <value>true</value>
</property>
<property>
  <name>hbase.regionserver.kerberos.principal</name>
  <value>hbase/_HOST@YOUR-REALM.COM</value>
</property>
<property>
  <name>hbase.regionserver.keytab.file</name>
  <value>/etc/hbase/conf/keytab.krb5</value>
</property>
<property>
  <name>hbase.master.kerberos.principal</name>
  <value>hbase/_HOST@YOUR-REALM.COM</value>
</property>
<property>
  <name>hbase.master.keytab.file</name>
  <value>/etc/hbase/conf/keytab.krb5</value>
</property>
<property>
<name>hbase.coprocessor.region.classes</name>
  <value>org.apache.hadoop.hbase.security.token.TokenProvider</value>
</property>
```


## Tips


1. Use `kdestroy` to destroy your local ticket cache. Do this to ensure that code
running locally is reading data in from a nominated keytab and not falling back
to the user's ticket cache.
1. VMs: Make sure the clocks between VM and host are in sync; it's easy for a VM clock
to drift when suspended and resumed.
1. VMs: Make sure that all hosts are listed in the host table, so that hostname lookup works.
1. Try to log in from a web browser without SPNEGO enabled; this will catch any WebUI
that wasn't actually authenticating callers.
1. Try to issue RPC and REST calls from an unauthenticated client, and from a user that is not granted
access rights.
1. YARN applications: verify that REST/Web requests against the real app URL (which can be
determined from the YARN application record), are redirected to the RM proxy (i.e. that
all GET calls result in a 30x redirect). If this does not take place, it means that the RM-hosted
SPNEGO authentication layer can be bypassed.
