# Reconnaissance

## 1. Ziel der Reconnaissance Phase

- Informationen über das Zielsystem sammeln
- Offene Ports identifizieren
- Laufende Services erkennen
- Potenzielle Angriffsvektoren ermitteln

---

## 2. Relevante Techniken & Vorgehensweisen

### Netzwerk-Scanning

- Ping Sweep zur Identifikation aktiver Hosts
- Port-Scanning (vollständig oder gezielt)
- Service-Version-Detection
- OS-Fingerprinting

### Service-Enumeration

- Banner Grabbing
- Versionserkennung laufender Dienste
- Identifikation von Web-Applikationen
- Directory/File Fuzzing

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- Standard-Ports: 21 (FTP), 22 (SSH), 80 (HTTP), 443 (HTTPS), 139/445 (SMB), 3306 (MySQL)
- Ubuntu/Debian typische Services
- Container-Umgebungen (Docker)
- Apache/Nginx Webserver
- Tomcat Application Server

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### Offene Ports

- FTP (21) - Anonymous Login, Credential Discovery
- SSH (22) - Brute Force, Key-based Auth
- DNS (53) - Zone Transfer, Subdomain Enumeration
- HTTP/HTTPS (80/443) - Web-Applikationen, CMS
- SMB (139/445) - Share Enumeration
- MySQL (3306) - Database Access
- Custom Ports (3000, 8080) - Application Servers

### Web-Services

- CMS (WordPress, Joomla, Navigate CMS, LimeSurvey, Grafana)
- Login-Portale
- File Upload Funktionen
- API Endpoints

---

## 5. Wichtige Befehle, Tools & Syntax

### Nmap

**Host Discovery (Ping Sweep)**


```bash
nmap -sn 192.168.xx.x/24
```

**Vollständiger Port-Scan**


```bash
nmap -p- -vv -T5 -sV 192.168.xx.x
sudo nmap -A -p- -T4 192.168.xx.x
nmap -Pn -sCV -p- 10.10.127.142 -oN nmap_full.txt
```

**Parameter-Erklärung:**

- `-sn`: Ping Scan (keine Port-Scans)
- `-p-`: Alle 65535 Ports
- `-sV`: Service Version Detection
- `-sC`: Standard Scripts
- `-A`: Aggressive Scan (OS Detection, Version Detection, Scripts, Traceroute)
- `-T4`/`-T5`: Timing (4=schnell, 5=sehr schnell)
- `-Pn`: Kein Ping (Ziel wird als online angenommen)
- `-oN`: Output in Datei
- `-vv`: Sehr verbose

**Spezifische Port-Scans**


```bash
nmap -p 22,80,443 192.168.xx.x
```

### Netcat (Banner Grabbing)


```bash
nc -v 192.168.xx.x 21
nc -v 192.168.xx.x 22
```

**Port-Scanning mit Netcat**


```bash
nc -zv 192.168.xx.x 1-1000
```

### Gobuster (Directory Fuzzing)


```bash
gobuster dir -u http://10.10.127.142 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
gobuster dir -u http://192.168.xx.x -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
gobuster dir -u http://blackpearl.tcm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### DirBuster (GUI Alternative)


```bash
dirbuster
```

- Target URL eingeben
- Threads: 200 ("go faster")
- Wordlist: `/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt`

### DNS Enumeration

**DNS Recon**


```bash
dnsrecon -r 127.0.0.0/24 -n 192.168.57.13 -d haxxor
```

**Showmount (NFS)**


```bash
showmount -e 192.168.xx.x
```

### Service-Spezifische Enumeration

**FTP**


```bash
ftp 192.168.xx.x
# Login: anonymous
# Password: (leer oder anonymous)
```

**SMB/Samba (NetExec/CrackMapExec)**


```bash
nxc smb 192.168.xx.x -u anonymous -p '' --ls
nxc smb 192.168.xx.x -u tom -p 'P@ssw0rd!' --shares
nxc ftp 192.168.xx.x -u anonymous -p '' --ls upload
```

**NFS**


```bash
nxc nfs 192.168.xx.x
```

**SSH**


```bash
ssh-keyscan 192.168.xx.x
```

**Rsync**


```bash
rsync --list-only rsync://10.10.112.78/
rsync -v rsync://10.10.112.78/backups/jenkins.tar .
```

### Service Version Check

**Grafana API Health Check**


```bash
curl http://10.10.118.75:3000/api/health
```

### Hash Identification


```bash
hash-identifier
# Hash eingeben wenn prompted
```

**Online Tools:**

- [https://hashes.com](https://hashes.com)

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard-Workflow

1. **Host Discovery**
    - Nmap Ping Sweep auf Subnetz
    - Aktive Hosts identifizieren
2. **Port Scanning**
    - Vollständiger Port-Scan (`-p-`)
    - Service-Version-Detection (`-sV`)
3. **Service Enumeration**
    - Banner Grabbing mit Netcat
    - Service-spezifische Tools nutzen
4. **Web-Enumeration (falls Port 80/443/8080 offen)**
    - Browser: Manuelle Inspektion
    - Directory Fuzzing (Gobuster/DirBuster)
    - Source Code Analyse
5. **Credential Discovery**
    - Default Credentials testen
    - Anonymous Access prüfen (FTP, SMB, NFS)
    - Config Files identifizieren
6. **Virtual Host Discovery (falls DNS/Web)**
    - `/etc/hosts` anpassen
    - Subdomains/VHosts testen

### Wichtige Hinweise

**Nmap Timing:**

- `-T5` ist laut, aber schnell
- `-T4` ist Standard für Labs
- In realen Umgebungen: `-T3` oder langsamer

**Wordlists:**

- `/usr/share/wordlists/dirb/common.txt` (klein, schnell)
- `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` (größer, umfassender)
- `/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt` (mittel)

**Output speichern:**

- Nmap: `-oN filename.txt`
- Immer Scan-Ergebnisse dokumentieren

**Hosts File Anpassung:**


```bash
sudo nano /etc/hosts
# Zeile hinzufügen:
192.168.xx.x blackpearl.tcm
```

---

## 7. Typische Findings & Fehler

### Häufige Findings

**Anonymous Access:**

- FTP: anonymous:anonymous
- SMB: anonymous Login ohne Passwort
- NFS: Shares ohne Authentifizierung

**Default Credentials:**

- Tomcat: admin:admin
- CMS: admin:admin123

**Interessante Directories:**

- `/admin`
- `/login`
- `/secret`
- `/backup`
- `/upload`
- `/feedback`
- `/academy`
- `/navigate`

**Konfigurationsdateien:**

- `config.xml` (Jenkins)
- `config.php` (Web-Apps)
- `wp-config.php` (WordPress)
- `tomcat-users.xml` (Tomcat)
- `grafana.db` (Grafana)

**Version Disclosure:**

- HTTP Headers
- API Endpoints (`/api/health`)
- Error Pages
- Source Code Kommentare

### Typische Fehler

**Unvollständige Scans:**

- Nur Standard-Ports scannen statt `-p-`
- Service-Detection vergessen (`-sV`)
- Zu schnelle Timings in realen Umgebungen

**Virtual Hosts übersehen:**

- Nicht nach Subdomains/VHosts suchen
- `/etc/hosts` nicht anpassen

**Output nicht speichern:**

- Scan-Ergebnisse nicht dokumentieren
- Keine Notizen während Enumeration

**Service-spezifische Enumeration vergessen:**

- Nach Port-Scan keine tiefere Analyse
- Tools wie `showmount`, `smbclient` nicht nutzen

**Credentials in Configs übersehen:**

- Keine Config-Files analysiert
- Environment Variables nicht geprüft
- Database-Credentials nicht gesucht
