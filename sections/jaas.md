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


# JAAS

JAAS is a nightmare from the Enterprise Java Bean era, one which surfaces from the depths to pull the unwary under. You can see its heritage whenever you search for documentation; it's generally related to managing the context of callers to EJB operations.


JAAS provides for a standard configuration file format for specifying a *login context*; how code trying to run in a specific context/role should login and authenticate.

As a single `jaas.conf` file can have multiple contexts, the same file can be used to configure the server and clients of a service, each with different binding information. Different contexts can have different login/auth mechanisms, including Kerberos and LDAP, so that you can even specify different auth mechanisms for different roles.

In Hadoop, the JAAS context is invariably Kerberos when it comes to talking to HDFS, YARN, etc.
However, if Zookeeper enters the mix, it may be interacted with differently —and so need a different JAAS context.

Fun facts about JAAS

1. Nobody ever mentions it, but the file takes backslashed-escapes like a Java string.
1. It needs escaped backlash directory separators on Windows, such as: `C:\\security\\krb5.conf`.
   Get that wrong and your code will fail with what will inevitably be an unintuitive message.
1. Each context must declare the authentication module to use.
   The kerberos authentication model on IBM JVMs is different from that on Oracle and OpenJDK JVMs.
   You need to know the target JVM for the context —or create separate contexts for the different JVMs.
1. The rules about when to use `=` within an entry, and when to complete an entry with a `;` appear to be:
start with the login module, one key=value line per entry, quote strings, finish with a `;`
within the same file.

Hadoop's UGI class will dynamically create a JAAS context for Hadoop logins, dynamically determining the name of the kerberos module to use. For interacting purely with HDFS and YARN, you may be able to avoid needing to know about or understand JAAS.

Example of a JAAS file valid for an Oracle JVM:
 

```
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useKeyTab=false
  useTicketCache=true
  doNotPrompt=true;
};
```


# Setting a JAAS Config file for a Java process

```
-Djava.security.auth.login.config=/path/to/server/jaas/file.conf
```

In Hadoop applications, this has to be set in whichever environment variable is picked up
by the command which your are invoking.
