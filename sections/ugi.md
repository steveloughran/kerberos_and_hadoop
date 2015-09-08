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
  
# UGI

> From the pictures I turned to the bulky, closely written letter itself; and for the next three hours was immersed in a gulf of unutterable horror. Where Akeley had given only outlines before, he now entered into minute details; presenting long transcripts of words overheard in the woods at night, long accounts of monstrous pinkish forms spied in thickets at twilight on the hills, and a terrible cosmic narrative derived from the application of profound and varied scholarship to the endless bygone discourses of the mad self-styled spy who had killed himself.

> HP Lovecraft [The Whisperer in Darkness](http://www.hplovecraft.com/writings/texts/fiction/wid.aspx), 1931

If there is one class guaranteed to strike fear into anyone with experience in Hadoop+Kerberos code it is `UserGroupInformation`, abbreviated to "UGI"

Nobody says `UserGroupInformation` out loud; it is the *him which must not be named* of the stack

## What does UGI do?

Here sre some of the things it can do

1. Handles the initial login process, using any environmental `kinit`-ed tokens or a keytab.
1. Spawn off a thread to renew the TGT
1. Support an operation for-on demand verification/re-init of kerberos tickets details before issuing a request.
1. Appear in stack traces which warn the viewer of security related trouble.


## UGI Strengths

* It's one place for almost all Kerberos/User authentication to live.
* Being fairly widely used, once you've learned it, your knowledge works through
the entire Hadoop stack.
 

## UGI Troublespots

* It's a singleton. Don't expect to have one "real user" per process. 
This does sort of makes sense. Even a single service has its "service" identity; as the 

* Once initialized, it stays initialized *and cannot be reset*.
This makes it critical to load in your configuration information including keytabs and principals,
before that first initialization of the UGI.
(There is actually a `UGI.reset()` call, but it is package scoped and purely to allow tests to
reset the information).
* UGI initialization can take place in code which you don't expect.
 A specific example is in the Hadoop filesystem APIs.
 Create a Hadoop filesystem instance and UGI is likely to be inited immediately, even if it is a local file:// reference.
 As a result: init before you go near the filesystem, with the principal you want.
* It has to do some low-level reflection-based access to Java-version-specific Kerberos internal classes.
This can break across Java versions, and JVM implementations. Specifically Java 8 has classes that Java 6 doesn't; the IBM JVM is very different.
* All its exceptions are basic `IOException` instancss, so hard to match on without looking at the text, which is very brittle.
* Some invoked operations are relayed without the stack trace (this should now be fixed).
* Diagnostics could be improved. (this is one of those British understatements, it really means "it would be really nice if you could actually get any hint as to WTF is going inside the class as otherwise you are left with nothing to go on other than some message that a user at a random bit of code wasn't authorized)

The issues related to diagnostics, logging, exception types and inner causes could be addressed. It would be nice to also have an exception cached at init time, so that diagnostics code could even track down where the init took place. Volunteers welcome. That said, here are some bits of the code where patches would be vetoed

* Replacing the current text of exceptions. We don't know what is scanning for that text, or what documents go with them.
Extending the text should be acceptable.
* All exceptions must remain subclasses of IOException.
* Logging must not leak secrets, such as tokens.


## Core UGI Operations


### `isSecurityEnabled()`

One of the foundational calls is the `UserGroupInformation.isSecurityEnabled()`

It crops up in code like this

    if(!UserGroupInformation.isSecurityEnabled()) {
        stayInALifeOfNaiveInnocence();
     } else {
        sufferTheEnternalPainOfKerberos();
     }

Having two branches of code, the "insecure" and "secure mode" is actually dangerous: the entire
security-enabled branch only ever gets executed when run against a secure Hadoop cluster

**Important**

*If you have separate secure and insecure codepaths, you must test on a secure cluster*
*alongside an insecure one. Otherwise coverage of code and application state will be*
*fundamentally lacking.*

*Unless you put in the effort, all your unit tests will be of the insecure codepath.*

*This means there's an entire codepath which won't get exercised until you run integration*
*tests on a secure cluster, or worse: until you ship.*

What to do? Alongside the testing, the other strategy is: keep the differences between
the two branches down to a minimum. If you look at what YARN does, it always uses
renewable tokens to authenticate communication between the YARN Resource Manager and
a deployed application. As a result, one codepath for token creation, while token propagation
and renewal is automatically tested on all applications.

Could your applications do the same? Certainly as far as token- and delegation-token based
mechanisms for callers to show that they have been granted access rights to a service.

### `getLoginUser()`

This returns the logged in user

    UserGroupInformation user = UserGroupInformation.getLoginUser();

If there is no current user --that is, the login process hasn't started yet,
this triggers the login and the starting of the background refresh thread.

This makes it a point where the security kicks in: all configuration resources
must be loaded in advance.

### `checkTGTAndReloginFromKeytab()`

If security is not enabled, this is a no-op.

If security is enabled, this will trigger a re-login if needed (which may fail,
of course).

*Important*: If the login fails, UGI will remember this and not retry until a time
limit has passed, even if other methods invoke the operation. The property
`hadoop.kerberos.min.seconds.before.relogin` controls this delay; the default is 60s.

What does that mean? A failure lasts for a while, even if it is a transient one. 

### `getCurrentUser()`

This returns the *current* user. 

The current user is not always the same as the logged in user; it changes
when a service performs an action on the user's behalf

### `doAs()`

## Environment variable-managed UGI Initialization

There are some environment variables which configure UGI.


| Environment Variable   | Meaning                   |
|----------------------------------------------------|----------------------------|
| HADOOP_PROXY_USER | identity of a proxy user to authenticate as |
| HADOOP_TOKEN_FILE_LOCATION | local path to a token file |

Why environment variables? They offer some features

1. Hadoop environment setup scripts can set them
1. When launching YARN containers, they may be set as environment variables.

As the UGI code is shared across all clients of HDFS and YARN; these environment
variables can be used to configure *any* application which communicates with Hadoop
services via the UGI-authenticated clients. Essentially: all Java IPC clients and
those REST clients using (the Hadoop-implemented REST clients)[web_and_rest.html].

## Debugging UGI

UGI supports low-level logging via the log4J log `org.apache.hadoop.security.UserGroupInformation`;
set the system property `HADOOP_JAAS_DEBUG=true` to have the JAAS context logging at
the debug level via some Java log API.

It's helpful to back this up with logging the `org.apache.hadoop.security.authentication`
package in `hadoop-auth`

```
log4j.logger.org.apache.hadoop.security.authentication=DEBUG
log4j.logger.org.apache.hadoop.security=DEBUG
```

