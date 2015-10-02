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

# Checklists

## All programs

[ ] Sets up security before accessing any Hadoop services, including FileSystem APIs

[ ] Sets up security after loading local configuration files, such as `core-site.xml`. Creating instances of HdfsConfiguration and YarnConfiguration will do this automatically.

[ ] Are tested against a secure cluster

## Hadoop RPC Service

[ ] Principal for Service defined. This is generally a configuration property.

[ ] `SecurityInfo` subclass written.

[ ] `META-INF/services/org.apache.hadoop.security.SecurityInfo` resource lists.

[ ] the `SecurityInfo` subclass written

[ ] `PolicyProvider` subclass written.

[ ] RPC server handed `PolicyProvider` subclass during setup.

[ ] Service verifies that caller has authorization for the action before executing it.

[ ] Service records authorization failures to audit log.
 
[ ] Service records successful action to audit log.

[ ] Uses `doAs()` to perform operations as the user making the RPC call.

## YARN Client/launcher

[ ] `HADOOP_USER` env variable set on AM launch context in insecure clusters, and in launched containers.

[ ] In secure cluster: all delegation tokens needed (HDFS, Hive, HBase, Zookeeper) created and added to launch context.

## YARN Application

[ ] Delegation tokens extracted and saved.

[ ] When launching containers, the relevant subset of delegation tokens are passed to the containers. (This normally omits the RM/AM token).

[ ] Container Credentials are retrieved in AM and containers.

[ ] Delegation tokens revoked during (managed) teardown.

## YARN Web UIs and REST endpoints
 
[ ] Primary Web server: `AmFilterInitializer` used to redirect requests to the RM Proxy.

[ ] Other web servers: a custom authentication strategy is chosen and implemented.
 
## Yarn Service

[ ] A strategy for token renewal is chosen and implemented

## Web Service

[ ] `AuthenticationFilter` added to web filter chain

[ ] Token renewal policy defined and implemented. (Look at `TimelineClientImpl` for an example of this)


## Clients

### All clients

[ ] Supports keytab login and calls `UserGroupInformation.loginUserFromKeytab(principalName, keytabFilename)` during initialization.

[ ] Issues `UserGroupInformation.getCurrentUser().checkTGTAndReloginFromKeytab()` call during connection setup/token reset. This is harmless on an insecure or non-keytab client.

[ ] Client supports Authentication Token option

[ ] Client supports Delegation Token option. (not so relevant for most YARN clients)

[ ] For Delegation-token authenticated connections, something runs in the background to regularly update delegation tokens.

[ ] Tested against secure clusters with user logged out (kdestroy).

[ ] Logs basic security operations at INFO, with detailed operations at DEBUG level.

### RESTful client

[ ] Jersey: URL constructor handles SPNEGO Auth

[ ] Code invoking Jersey Client reacts to 401/403 exception responses when using Authentication Token by deleting creating a new Auth Token and re-issuing request. (this triggers re-authentication)

