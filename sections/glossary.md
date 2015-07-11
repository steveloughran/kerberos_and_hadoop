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
