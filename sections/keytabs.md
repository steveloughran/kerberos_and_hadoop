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

# Keytabs

Keytabs are critical for secure Hadoop clusters, as they allow the services to be launched
without prompts for passwords


## Creating a Keytab

If your management tools sets up keytabs for you: use it.

```bash

kadmin.local

ktadd -k zk.service.keytab -norandkey zookeeper/devix@COTHAM 
ktadd -k zk.service.keytab -norandkey zookeeper/devix.cotham.uk@COTHAM
exit
```

and of course, make it accessible

```bash
chgrp hadoop zk.service.keytab
chown zookeeper zk.service.keytab
```

check that the user can login

```bash
# sudo -u zookeeper klist -e -kt zk.service.keytab
# sudo -u zookeeper kinit -kt zk.service.keytab zookeeper/devix.cotham.uk
# sudo -u zookeeper klist
```

### Keytab Expiry

Keytabs expire

That is: entries in them have a limited lifespan (default: 1 year)

This is actually a feature —it limits how long a lost/stolen keytab can have access to the system.

At the same time, it's a major inconvenience as (a) the keytabs expire and (b) it's never
immediately obvious why your cluster has stopped working.

### Keytab security

Keytabs are sensitive items. They need to be treated as having all the access to the data of that principal

### Keytabs and YARN applications


