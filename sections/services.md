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
  
# Securing Hadoop Services

## Proxy Users and superuser services

Supporting Proxy users is covered in the hadoop docs
[Proxy user - Superusers Acting On Behalf Of Other Users](http://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-common/Superusers.html)
 
In an insecure user, the proxy user creation is automatic: services act on behalf of whoever
they claim to be.

In a secure cluster, they must have (how??) received the delegation token for
the user.


Note that in Hadoop 2.6/2.7 the Filesystem cache creates a new filesystem instance for the given user, even
if there is an FS client for that specific filesystem already in the cache. (VERIFY)

