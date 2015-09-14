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
  
# YARN  and YARN Applications
 
YARN applications are somewhere where Hadoop authentication becomes some of its most complex.

Anyone writing a YARN application will encounter Hadoop security, and will end up spending
time debugging the problems. This is "the price of security".

## YARN Service security

YARN Resource Managers (RM) and Node Managers (NM) perform work on behalf of the user.

The NM's

1. `Localize` resources: Download from HDFS or other filesystem into a local directory. This
is done using the delegation tokens attached to the container launch context. (For non-HDFS
resources, using other credentials such as object store login details in cluster configuratio
files)

1. Start the application as the user.

## Securing YARN Application Web UIs and REST APIs

YARN provides a straightforward way of giving every YARN application SPNEGO authenticated
web pages: it implements SPNEGO authentication in the Resource Manager Proxy. 
YARN web UI are expected to load the AM proxy filter when setting up its web UI; this filter
will redirect all HTTP Requests coming from any host other than the RM Proxy hosts to an
RM proxy, to which the client app/browser must re-issue the request. The client will authenticate
against the principal of the RM PRoxy (usually `yarn`), and, once authenticated, have its
request forwared.

As a result, all client interactions are SPNEGO-authenticated, without the YARN application
itself needing any kerberos principal for the clients to authenticate against.

Known weaknesses in this approach are

1. As calls coming from the proxy hosts are not redirected, any application running
on those hosts has unrestricted access to the YARN applications. This is why in a secure cluster
the proxy hosts must run on cluster nodes which do not run end user code (i.e. not run YARN
NodeManagers and hence schedule YARN containers).
1. The HTTP requests between proxy and YARN RM Server are not encrypted.

## Securing YARN Application REST APIs

YARN REST APIs running on the same port as the registered web UI of a YARN application are
automatically authenticated via SPNEGO authentication in the RM proxy.
 
Any REST endpoint (and equally, any web UI) brought up on a different port does not
support SPNEGO authentication unless implemented in the YARN application itself.

## Strategies for token renewal on YARN services


### Keytabs for AM and containers

A keytab is provided for the application. This can be done by:

1. Installing it in every cluster node, then providing the path
to this in a configuration directory. The keytab must be in a secure directory path, where
only the service (and other trusted accounts) can read it.

1. Including the keytab as a resource for the container, relying on the Node Manager localizer
to download it from HDFS and store it locally. This avoids the administration task of
installing keytabs for specific services. It does require the client to have access to the keytab
and as it is uploaded to the distributed filesystem, must be secured through the appropriate 
path permissions. 

This is the strategy adopted by Apache Slider (incubating). Slider also pushes out specified
keytabs for deployed applications such as HBase, with the Application Master declaring the
HDFS paths to them in its Container Launch Requests.

### AM keytab + renewal and forwarding of Delegation Tokens to containers

The Application Master is given the path to a keytab (usually a client-uploaded localized resource),
so can stay authenticated with Kerberos. Launched containers are only given delegation tokens.
Before a delegation token is due to expire, the processes running in the containers must request new
tokens from the Application Master. Obviously, communications between containers and their Application
Master must themselves be authenticated, so as to prevent malicious code from requesting the containers
from an Application Master.

This is the strategy used by Spark 1.5+. Communications between container processes and the AM
is over HTTPS-authenticated Akka channels.


### Client-side push of renewed Delegation Tokens

This strategy may be the sole one acceptable to a strict operations team: a client process
running on an account holding a kerberos TGT negotiates with all needed cluster services
for delegation tokens, tokens which are then pushed out to the Application Master via
some RPC interface.

This does require the client process to be re-executed on a regular basis; a cron or OOzie job
can do this.
