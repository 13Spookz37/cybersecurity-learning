# Windows Reconnaissance


## Ziel der Reconnaissance Phase

- Identifikation offener Ports und laufender Dienste
- Ermittlung der Betriebssystemversion und des Hostnamens
- Erkennung von Domain-Controllern und Active Directory-Strukturen
- Auffinden von Web-Services, SMB-Shares und anderen Angriffsflächen
- Sammlung von Informationen über Benutzer, Gruppen und Systeme

---

## Relevante Techniken & Vorgehensweisen

### Netzwerk-Scanning

**Host Discovery:**


```bash
# ICMP-Test
ping <TARGET_IP>

# ARP-Scan (lokales Netzwerk)
sudo arp-scan -l

# Nmap Host Discovery
nmap -sn 192.168.xx.x/24
```

**Port-Scanning:**


```bash
# Standard-Scan
nmap <TARGET_IP>

# Umfassender Scan
nmap -A -p- -T4 <TARGET_IP>
nmap -sC -sV <TARGET_IP>
nmap -A -sC -sV -p- <TARGET_IP>

# Aggressive Scan mit allen Ports
sudo nmap -A -p- 192.168.xx.x -T4
sudo nmap -Pn -sS -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,5986,9389 <TARGET_IP>

# Spezifische Ports
sudo nmap -sC -sV 192.168.xx.x/24 -p 445,139,88,389,636,80,443,22,3389 --open
```

**Vulnerability Scanning:**


```bash
# SMB Vulnerability Scripts
nmap --script smb-vuln* -p 139,445 <TARGET_IP>
```

### Web-Enumeration

**Web-Zugriff:**


```bash
# Browser
http://<TARGET_IP>
http://<TARGET_IP>:8080
http://<TARGET_IP>:49663

# cURL
curl http://<TARGET_IP>
curl http://<TARGET_IP>/README.txt
```

**Directory Enumeration:**


```bash
# Gobuster
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Dirbuster (GUI)
dirbuster &
# Target URL: http://<TARGET_IP>:80/
# Wordlist: small.txt oder directory-list-2.3-medium.txt

# ffuf
ffuf -u http://<TARGET_IP>/dev/FUZZ -w /usr/share/wordlists/dirb/common.txt

# dirb
dirb http://<TARGET_IP>

# Dirsearch
dirsearch -u http://<TARGET_IP>
```

**Web Vulnerability Scanning:**


```bash
# Nikto
nikto -h <TARGET_IP>
nikto -h http://<TARGET_IP>:<PORT> -id bob:bubbles
```

**Web Application Enumeration:**


```bash
# WordPress
wpscan --url http://blog.thm --enumerate u
wpscan -U users.txt -P /usr/share/wordlists/rockyou.txt --url http://blog.thm

# PHPInfo
curl http://<TARGET_IP>/dashboard/phpinfo.php

# Joomla Version
curl http://<TARGET_IP>/README.txt
```

### SMB-Enumeration

**Share-Auflistung:**


```bash
# SMBClient
smbclient -L \\<TARGET_IP>\\
smbclient -L \\192.168.xx.x\\
smbclient \\\\<TARGET_IP>\\ADMIN$
smbclient \\\\<TARGET_IP>\\IPC$
smbclient //<TARGET_IP>/Trainees
smbclient //blog.thm/BillySMB

# Mit Credentials
smbclient //192.168.xx.x/Notes -U trainee%trainee

# NetExec (CrackMapExec)
nxc smb <TARGET_IP> -u 'badguy' -p '' --shares
nxc smb 192.168.xx.x -u 'trainee' -p 'trainee' --shares
crackmapexec smb <TARGET_IP>

# Nmap SMB Scripts
nmap --script smb-enum-shares <TARGET_IP>
```

**Metasploit SMB-Module:**


```bash
msfconsole
search SMB
search auxiliary/scanner/smb
use 16
show options
set RHOSTS <TARGET_IP>
run
```

**SMB Security Check:**


```bash
crackmapexec smb <TARGET_IP>
```

### LDAP-Enumeration

**LDAP-Abfragen:**


```bash
# Anonymous Bind
ldapsearch -x -H ldap://<TARGET_IP> -b "DC=baby,DC=vl"
ldapsearch -H ldap://<TARGET_IP> -x -s base namingcontexts

# User Enumeration
ldapsearch -H ldap://<TARGET_IP> -x -b "DC=lab,DC=trusted,DC=vl" "(objectClass=user)" sAMAccountName
```

**Enum4linux:**


```bash
enum4linux-ng -A <TARGET_IP>
```

### RPC-Enumeration

**RPCClient:**


```bash
rpcclient -U "" -N <TARGET_IP>
```

**SID Bruteforcing:**


```bash
impacket-lookupsid anonymous@<TARGET_IP>
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@<TARGET_IP>
```

### NFS-Enumeration

**NFS Shares:**


```bash
# Exports anzeigen
showmount -e <TARGET_IP>

# Mounten
sudo mkdir /mnt/dev
sudo mount -t nfs <TARGET_IP>:/srv/nfs /mnt/dev
sudo mount -t nfs -o vers=3,nolock <TARGET_IP>:/opt/share /tmp/mount

# Navigieren
cd /mnt/dev
ls
```

### DNS-Enumeration

**Hosts-Datei aktualisieren:**


```bash
sudo nano /etc/hosts
echo "<TARGET_IP> domain.vl dc.domain.vl" | sudo tee -a /etc/hosts
echo "192.168.xx.x blog.thm" >> /etc/hosts
```

### Kerberos-Enumeration

**Kerbrute:**


```bash
kerbrute userenum --dc <TARGET_IP> -d lab.trusted.vl /usr/share/seclists/Usernames/Names/names.txt
```

### Credentials & Hashes

**Passwort-Listen:**


````bash
# Rockyou herunterladen
sudo wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt -O /usr/share/wordlists/rockyou.txt
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows-Version & Dienste

**Typische Ports:**
- 22: SSH
- 53: DNS
- 80/443: HTTP/HTTPS
- 88: Kerberos
- 135: MSRPC
- 139/445: SMB
- 389/636: LDAP/LDAPS
- 464: Kerberos Password Change
- 593: RPC over HTTP
- 1234: Tomcat (ungewöhnlich)
- 3000: Gitea
- 3268/3269: Global Catalog LDAP
- 3389: RDP
- 5357: WS-Discovery
- 5985/5986: WinRM
- 8009: AJP13
- 8080: HTTP Alternative
- 9389: AD Web Services

**Windows Server Identifikation:**
- Windows Server 2022 (Build 20348)
- Windows Server 2008 R2 SP1
- Windows 7 Ultimate 7601 Service Pack 1

### Domain Controller Erkennung

**Nmap-Ausgabe analysieren:**
```
PORT    STATE SERVICE
53/tcp  open  domain
88/tcp  open  kerberos-sec
135/tcp open  msrpc
139/tcp open  netbios-ssn
389/tcp open  ldap
445/tcp open  microsoft-ds
636/tcp open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
````

**Domain-Informationen:**

- Domain: retro.vl, baby.vl, hybrid.vl, trusted.vl
- NetBIOS Domain Name: RETRO, BLOG, ESCAPE
- DNS Domain Name: Escape, retro2.vl
- Computer Name: DC, ESCAPE, BLN01

---

## Typische Angriffsflächen

### SMB

- Anonymous Access auf Shares
- Readable Shares: Notes, Trainees, BillySMB
- Credentials in Share-Dateien (Important.txt, ToDo.txt)
- Vorkonfigurierte Computer-Accounts (BANKING$)

### Web-Services

- IIS Default Pages
- Apache/Tomcat Manager
- Jenkins (Port 8080)
- WordPress
- Joomla
- Gitea (Port 3000)
- Roundcube Webmail
- phpinfo() Exposure
- LFI Vulnerabilities
- Directory Listing

### LDAP

- Anonymous Bind möglich
- Klartext-Passwörter in Beschreibungen
- User Enumeration

### NFS

- World-readable Exports
- Backup-Dateien
- Passwort-geschützte Archive

### Credentials in Dateien

- `Important.txt`, `ToDo.txt`, `README.txt`
- Konfigurationsdateien (config.xml, profiles.xml)
- Backups (backup.tar.gz, save.zip)
- Datenbanken (staff.accdb)

---

## Wichtige Befehle & Syntax

### Nmap Parameter


```bash
-A          # Aggressive Scan (OS detection, version detection, script scanning)
-sC         # Default Scripts
-sV         # Service Version Detection
-p-         # Alle 65.535 Ports scannen
-T4         # Fast Timing Template
-Pn         # Kein Ping (bei Firewall)
-sS         # SYN Stealth Scan
--open      # Nur offene Ports
--script    # Spezifische Scripts
```

### SMBClient


```bash
-L          # List Shares
-N          # No Password
-U          # Username
```

### Gobuster


````bash
dir         # Directory Mode
-u          # Target URL
-w          # Wordlist
-e          # Extensions (php,html,txt)
```

### Dirbuster GUI

- "Go Faster" Option aktivieren
- Wortlisten: small.txt, directory-list-2.3-medium.txt

---

## Reihenfolge & Abhängigkeiten

### Empfohlene Vorgehensweise

1. **ICMP/ARP-Test** → Erreichbarkeit prüfen
2. **Nmap Port-Scan** → Offene Ports identifizieren
3. **Service-Enumeration** → Versionen ermitteln
4. **SMB/LDAP/NFS** → Anonymen Zugriff testen
5. **Web-Enumeration** → Directories/Dateien finden
6. **Credential Gathering** → Passwörter sammeln
7. **Version Research** → Bekannte CVEs suchen

### Abhängigkeiten

- **DNS-Auflösung erforderlich:** Domain-Namen in `/etc/hosts` eintragen
- **SMB Signing:** Beeinflusst Relay-Angriffe
- **Anonymous Access:** Basis für weitere Enumeration
- **Versionserkennung:** Kritisch für Exploit-Suche

---

## Typische Findings

### Credentials aus Notes

**Trainee-Accounts:**
```
trainee:trainee
```

**MySQL:**
```
root:SuperSecureMySQLPassw0rd1337.
```

**Windows-Benutzer:**
```
User:Password123!
Administrator:Password456!
butler:JeNkIn5@44
administrator:A%rc!BcA!
jenkins:jenkins
jeanpaul:I_love_java
john:TwoCows2
bob:bubbles
kwheel:cutiepie1
```

**Domain-Accounts:**
```
Teresa.Bell:BabyStart123!
Caroline.Robinson:BabyStart123! (Password must change)
peter.turner@hybrid.vl:PeterIstToll!
admin@hybrid.vl:Duckling21
cpowers:322db798a55f85f09b3d61b976a13c43 (Hash)
````

### Versionsinformationen

**Vulnerable Software:**

- Apache 1.3.20
- Samba 2.2.1a
- OpenSSL 0.9.6b
- mod_ssl 2.8.4
- Joomla 3.7.0
- WordPress 5.0
- Apache Tomcat/7.0.88
- Jenkins (jenkins:jenkins)
- PDF24 Creator 11.15.1

### Wichtige Dateien

**Web:**

- /README.txt, /LICENSE.txt
- /robots.txt
- /administrator/ (Joomla)
- /wp-admin/ (WordPress)
- /manager/html (Tomcat)
- config.yml, configuration.php, wp-config.php

**SMB:**

- Important.txt, ToDo.txt
- profiles.xml (mRemoteNG)
- config.xml
- Alice-White-Rabbit.jpg, tswift.mp4, check-this.png

**NFS:**

- save.zip (Passwort: java101)
- id_rsa (SSH-Key)
- todo.txt
- backup.tar.gz

### Domain-Struktur

**Computer-Accounts:**

- BANKING$ (alter Pre-created Account)
- MAIL01$

**Gruppen:**

- HelpDesk
- Domain Computers
- Domain Admins

**SIDs:**

- Child Domain: S-1-5-21-2241985869-2159962460-1278545866
- Parent Domain: S-1-5-21-3576695518-347000760-3731839591

### Ports & Services

**Häufige Konfigurationen:**

- Port 1234: Tomcat Manager
- Port 3000: Gitea
- Port 8080: Jenkins, PHP, BoltWire
- Port 49663: HTTP (SMB-Share über Web)

---

## Typische Fehler

### Netzwerk

- Firewall blockiert Ping → `-Pn` verwenden
- Nicht alle Ports gescannt → `-p-` nutzen
- DNS nicht konfiguriert → `/etc/hosts` aktualisieren

### SMB

- KEX-Algorithmus-Mismatch bei SSH
- SMB Signing required → Relay-Angriffe nicht möglich
- Credentials funktionieren nicht → Port/Service prüfen

### Web

- Standard-Wordlists zu klein → Größere Listen verwenden
- Spezielle Ports vergessen → 8080, 3000, 49663 prüfen
- Extensions nicht angegeben → php, html, txt testen

### Enumeration

- Oberflächliche Scans → Gründliche Enumeration erforderlich
- `/administrator/`, `/admin/` nicht geprüft
- Git-History nicht untersucht → Alte Commits enthalten Secrets
- NFS-Mounts nicht getestet
- LDAP Anonymous Bind nicht versucht
