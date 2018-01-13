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
  
  
> Johansen, thank God, did not know quite all, even though he saw the city and the Thing, but I shall never sleep calmly again when I think of the horrors that lurk ceaselessly behind life in time and in space, and of those unhallowed blasphemies from elder stars which dream beneath the sea, known and favoured by a nightmare cult ready and eager to loose them upon the world whenever another earthquake shall heave their monstrous stone city again to the sun and air.

> HP Lovecraft [The Call of Cthulhu](http://www.hplovecraft.com/writings/texts/fiction/cc.aspx), 1926

# Java and JDK Versions

Kerberos support is built into the Java JRE. It comes in two parts

1. The public APIs with some guarantee of stability at the class/method level, along with assertions of functionality behind those classes and methods.
2. The internal implementation classes which are needed to do anything sophisticated.

The Hadoop UGI code uses part (2), the internal `com.sun` classes, as the public APIs
are too simplistic for the authentication system.

This comes at a price

1. Different JRE vendors (e.g IBMs) use different implementation classes, so will not work.
2. Different Java versions can (and do) change those implementation classes.

Using internal classes is one of those "don't do this your code will be unreliable" rules; it wasn't something done lightly. In its defence, (a) it was needed and (b) it turns out that the public APIs are brittle across versions and JDKs too, so we haven't lost as much as you'd think.

### Key things to know

* Hadoop is built and tested against the Oracle JDKs
* Open JDK has the same classes and methods, so will behave consistently; it's tested against too.
* It's left to the vendors of other JVMs to test their code; the patches are taken on trust.
* The Kerberos internal access usually needs fixing across Java versions. This means secure Hadoop clusters absolutely require the Java versions listed on the download requirements.
* Releases within a Java version may break the internals and/or the public API's behaviour.
* If you want to see the details of Hadoop's binding, look in `org.apache.hadoop.security.authentication.util.KerberosUtil` in the `hadoop-auth` module.

To put it differently: 

## The Hadoop security team is always scared of a new version of Java

