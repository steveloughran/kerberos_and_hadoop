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

# Hadoop's support for Kerberos

Hadoop can use Kerberos to authenticate users, and processes running within a
Hadoop cluster acting on behalf of the user. It is also used to authenticate services running
within the Hadoop cluster itself -so that only authenticated HDFS Datanodes can join the HDFS
filesystem, that only trusted Node Managers can heartbeat to the YARN Resource Manager and
receive work.

* The exact means by which all this is done is one of the most complicated pieces of code to span the
entire Hadoop codebase.*
 
Users of Hadoop do not need to worry about the implementation details, and, ideally, nor should
the operations team.

Developers of core Hadoop code, anyone writing a YARN application, and anyone writing code
to interact with a Hadoop cluster and applications running in it *do need to know those details*.

This is what this book attempts to cover.

## Why do they inflict so much pain on us?

Before going in there, here's a recurring question: why? Why Kerberos and not, say some
SSL-certificate like system? Or OAuth?

Kerberos was written to support centrally managed accounts in a local area network, one in
which adminstrators manage individual accounts. This is actually much simpler to manage than
PKI-certificate based systems: look at the effort it takes to revoke a certificate in a browser.
