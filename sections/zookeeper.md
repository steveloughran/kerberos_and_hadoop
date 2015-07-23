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
  
# Zookeeper

Apache Zookeeper uses SASL to authenticate callers. 

Other than SASL, its access control is all based around secrets which are shared between client and server, and sent over the (unencrypted) channel. This means they cannot be relied upon to securely identify callers.

## Enabling SASL in ZK

SASL is enabled in ZK by setting a system property. While adequate for a server, it's less than convenient when using ZK in an application as it means something very important: you cannot have a non-SASL and a SASL ZK connection at the same time. (you could create one connection, change the system properties and then create the next, but as Apache Curator doesn't do this, and everyone sensible uses Curator to handle transient disconnections and ZK node failover, this isn't practicable). Someone needs to fix this.


 