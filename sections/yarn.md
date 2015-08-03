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

## Strategies for token renewal on YARN services

### Client-side push of renewed Delegation Tokens

### Keytabs for containers

### AM keytab + renewal and forwarding of Delegation Tokens to containers

