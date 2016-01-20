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

# Tales of Terror

The following are all true stories. We welcome more submissions of these stories, which will
all be repeated anonymously.


## The Zookeeper's Birthday Present


A client program could not work with zookeeper: the connections were being broken. But it
was working for everything else.

The cluster was one year old that day.

It turns out that ZK reacts to an auth failure by logging something in its logs, and breaking
the client connection —without any notification to the client. Rather than a network problem
(initial hypothesis), this was discovered to be an HDFS problem.

When a Kerberos keytab is created, the entries in it have a lifespan. The default value is one
year. This was its first birthday, hence ZK wouldn't trust the client.

