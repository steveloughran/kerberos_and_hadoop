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
# Glossary

 
* KPW - Kerberos Password
Used to encrypt and validate session keys

* TGT - Kerberos Ticket Granting Ticket
 A special KST granting user access to TGS
 Stored in user's keytab via kinit or windows login 

*  KST - Kerberos Service Ticket

*  KST[P,S] - A KST granting principal P access to service S
*  DT - Delegation Token
*  DT[P,R] - Allows holder to impersonate principal P and renew DT with service R
*  JT - Job Token
*  A secure random value authenticating communication between JT and TT about a given task
*  BT - Block Access Token
*  BT[P,B] - A BT granting principal P access to block B
