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
  
# SPNEGO

SPNEGO is the acronym of the protocol by which HTTP clients can authenticate with a web site using Kerberos. This allows the client to identify and authenticate itself to a web site or a web service.
SPNEGO is supported by

* the standard browsers, to different levels of pain of use
* `curl` on the command line
* `java.net.URL` in Java7+

The final point is key: it can be used programmatically in Java, so used by REST client applications to authenticate with a remote Web Service.

### CAUTION

Apache Http Components do not support SPNEGO. As the documentation says "try it and see" {cite}.
Exactly how the Java runtime implements its SPNEGO authentication is a mystery to all. Unlike, say Hadoop IPC, where the entire authentication code has been implemented by people whose email addresses you can identify from the change log and so ask hard questions, what the JDK does is a black hole.

## Configuring Firefox to use SPNEGO

Firefox is the easiest browser to set up with SPNEGO support, as it is done in about:config and then persisted
Here are the settings for a local VM, a VM which has an entry in the /etc/hosts:
192.168.1.134 devix.cotham.uk devix

This hostname is then listed in firefox's config as a URL to trust.

![firefox spnego][../images/firefox_spnego_setup.png]

## Chrome and SPNEGO

Historically, Chrome needed to be configured on the command line to use SPNEGO, which was complicated to the point of unusability.

Fortunately, there is a better way, [Chromium Policy Templates](https://www.chromium.org/administrators/policy-templates).

See [Google Chrome, SPNEGO, and WebHDFS on Hadoop](http://www.ghostar.org/2015/06/google-chrome-spnego-and-webhdfs-on-hadoop/)


## Adding your own custom webauth initializer

Many large organizations implement their oi