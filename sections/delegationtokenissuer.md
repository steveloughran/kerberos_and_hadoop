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

# `interface DelegationTokenIssuer`

This interface was extracted from the delegation token methods of the `FileSystem`
class; it was extracted so that it can also be used to create delegation
tokens from other sources, specifically the KMS Key manager.



### `String getCanonicalServiceName()`

The method `getCanonicalServiceName()` is used to return a string for the filesystem
by which a Delegation Token issued by the filesystem may be indexed within
a map of `ServiceName` to `Token`.

If non-null, it must be unique to the extent that: 

_all filesystem instances or
other token issuers which declare their canonical service name to be identical
MUST must be able to use the same marshalled delegation token to authenticate._

Generally the service name is the URI of the filesystem or other service.

For any filesystem or service where the port is used to distinguish the service identity,
the port MUST be included in the canonical service name. e.g. `hdfs://namenode:8088/`. 

For object stores accessed over HTTP or HTTPS, the port is generally excluded

`s3a://bucket1/`



*Note* `AbstractFileSystem.getUriDefaultPort()` is used in 
the `AbstractFileSystem.checkPath()` to verify that a path passed in to a `FileContext`
API call applies to the filesystem being invoked. 

If the canonical URI of a filesystem includes a port, the same value should be 
be returned inthe `getDefaultPort()`


The implemntation of the method in `FileSystem` is: 

```java
  public String getCanonicalServiceName() {
    return (getChildFileSystems() == null)
      ? SecurityUtil.buildDTServiceName(getUri(), getDefaultPort())
      : null;
  }
```
That is: if there are no nested child filesystems, then the Canonical Service
URI is derived from `SecurityUtil.buildDTServiceName(getUri(), getDefaultPort())`



If a filesystem returns `null` from `getCanonicalServiceName()`, it is
declaring that it does not issue delegation tokens, that is `getDelegationToken()`
will also return `null`.

### `Token<?> getDelegationToken(renewer)`

Request a delegation token from the filesystem/delegation token issuer.


### `Token<?>[] addDelegationTokens` 


###   `DelegationTokenIssuer[] getAdditionalTokenIssuers()`


Return a possibly empty array of other token issuers, or `null`.

The token collector is expected to recursively collect all tokens served
up by issuers listed. This is to support filesystems which contain one
or more nested filesystem through mount points, and services such as key
management associated with the store.

The return value *MUST NOT* contain a reference to the current issuer: callers
do not check for this when recursing through the tree of token issuers.
