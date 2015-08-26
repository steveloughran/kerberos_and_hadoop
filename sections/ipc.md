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


> Man's respect for the imponderables varies according to his mental constitution and environment. Through certain modes of thought and training it can be elevated tremendously, yet there is always a limit.

> *[At the Root](https://en.wikisource.org/wiki/At_the_Root), HP Lovecraft, 1918.*

# Hadoop IPC Security

The Hadoop IPC system handles Kerberos ticket and Hadoop token authentication automatically.

1. The identity of the principals of services are configured in the hadoop site configuration
files.
1. Every IPC services uses java annotations, a metadata resource file and a custom `SecurityInfo`
subclass to define the security information of the IPC, including the key in the configuration
used to define the principal.
1. If a caller making a connection has a valid token (auth or delegate) it is used
to authenticate with the remote principal.
1. If a caller lacks a token, the Hadoop ticket will be used to acquire an authentication
token.
1. Applications may explicitly request delegation tokens to forward to other processes.
1. Delegation tokens are renewed in a background thread (which?).

## Adding a new IPC interface to a Hadoop Service/Application

This is "fiddly". It's not impossible, it just involves effort.

In its favour: it's a lot easier than SPNEGO.

### `SecurityInfo` subclass

Every exported RPC service will need its own extension of the `SecurityInfo` class, to provide two things:

1. The name of the principal to use in this communication
1. The token used to authenticate ongoing communications.

### `PolicyProvider` subclass

A `PolicyProvider` subclass. This is used to inform the RPC infrastructure of the ACL policy: who may talk to the service. It must be explicitly passed to the RPC server

		rpcService.getServer()
		  .refreshServiceAcl(serviceConf, new MyRPCPolicyProvider());

### `SecurityInfo` resource file

The resource file `META-INF/services/org.apache.hadoop.security.SecurityInfo` lists all RPC APIs which have a matching SecurityInfo subclass in that JAR.

		org.apache.example.appmaster.rpc.RPCSecurityInfo

The RPC framework will read this file and build up the security information for the APIs (server side? Client side? both?)


