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
  
# Hadoop IPC Security

## Adding a new IPC interface to a Hadoop Service/Application

This is "fiddly". It's not impossible, it just involves effort. In its favour: it's a lot easier than SPNEGO.

### `SecurityInfo` subclass

Every exported RPC service will need its own extension of the `SecurityInfo` class, to provide two things:

1. The name of the principal to use in this communication
1. The token used to authenticate ongoing communications.

### `PolicyProvider` subclass

A `PolicyProvider` subclass. This is used to inform the RPC infrastructure of the ACL policy: who may talk to the service. It must be explicitly passed to the RPC server

		rpcService.getServer()
		  .refreshServiceAcl(serviceConf, new MyRPCPolicyProvider());

### SecurityInfo resource file

The resource file `META-INF/services/org.apache.hadoop.security.SecurityInfo` lists all RPC APIs which have a matching SecurityInfo subclass in that JAR.

		org.apache.example.appmaster.rpc.RPCSecurityInfo

The RPC framework will read this file and build up the security information for the APIs (server side? Client side? both?)



----
 