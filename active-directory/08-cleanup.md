# Cleanup

# Ziel der Cleanup Phase

- Alle erstellten Backdoor-Accounts entfernen
- Hochgeladene Tools und Payloads löschen
- Malicious Files (LNK, DLL, Scripts) entfernen
- Laufende Angriffs-Prozesse beenden
- Network Tunnels und Proxies schließen
- Responder Konfiguration zurücksetzen
- System in ursprünglichen Zustand versetzen

---

## Backdoor User entfernen

### Domain User löschen

cmd

```cmd
# Domain User entfernen
net user backdoor /delete /domain
net user hawkeye /delete /domain
net user harry /delete /domain
```

### Automatisch erstellte User (mitm6)


```bash
# User aus mitm6 Attack identifizieren
cat lootme/domain_users.html
# Nach Benutzern mit zufälligen Namen suchen (z.B. hSnKXmKTJa)
```

cmd

```cmd
# Via Evil-WinRM oder PSExec auf DC
net user hSnKXmKTJa /delete /domain
```

---

## Hochgeladene Tools entfernen

### Mimikatz Files löschen

cmd

```cmd
# Mimikatz und zugehörige Dateien
del C:\Windows\Temp\mimikatz.exe
del C:\Windows\Temp\mimidrv.sys
del C:\Windows\Temp\mimilib.dll
del C:\Windows\Temp\mimispool.dll
```

### Verification

cmd

```cmd
# Prüfen ob Files gelöscht
dir C:\Windows\Temp\mimi*.*
```

---

## Malicious Files entfernen

### LNK Files löschen

cmd

```cmd
# Malicious LNK Files entfernen
del C:\important_document.lnk
```

### PowerShell Scripts entfernen

cmd

```cmd
# Falls PowerShell Scripts erstellt wurden
del C:\Users\*\*.ps1
del C:\Temp\*.ps1
```

### Webshells entfernen


```bash
# Via Shell auf kompromittiertem Webserver
rm /var/www/html/dev/back.php
rm C:\xampp\htdocs\dev\back.php
```

---

## Payloads entfernen

### Meterpreter DLL löschen


```bash
# SMB Server beenden
pkill -f smbserver.py

# Lokale Payload löschen
rm shell.dll
```

### Auf Target System

cmd

```cmd
# Falls DLL auf Target kopiert wurde
del C:\Windows\Temp\shell.dll
del C:\Temp\*.dll
```

---

## Angriffs-Prozesse beenden

### Responder beenden


```bash
# Responder Prozess killen
sudo pkill -f responder
```

### mitm6 beenden


```bash
# mitm6 Prozesse beenden
sudo pkill mitm6

# ntlmrelayx beenden
pkill -f ntlmrelayx
pkill -f impacket-ntlmrelayx
```

### Metasploit Sessions schließen


```bash
# In Meterpreter
exit

# In msfconsole
sessions -K    # Alle Sessions beenden
exit
```

---

## Network Tunnels beenden

### SSH Tunnels schließen


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
# SSHuttle Prozesse killen
sudo pkill sshuttle

# Routing wird automatisch wiederhergestellt
```

### Tunnel Status verifizieren


```bash
# Prüfen ob Tunnels noch laufen
netstat -tlnp | grep 9050
ps aux | grep ssh
ps aux | grep sshuttle
```

---

## Shell Sessions beenden

### Evil-WinRM beenden


```bash
# In Evil-WinRM Session
exit
```

### Impacket Shells beenden


```bash
# In PSExec/WMIExec/SMBExec
exit
```

### Interactive SMB Relay beenden


```bash
# nc/telnet Session beenden
exit

# ntlmrelayx beenden
pkill -f ntlmrelayx
```

---

## Responder Configuration zurücksetzen

### Original Config wiederherstellen


```bash
# Responder Config editieren
sudo nano /etc/responder/Responder.conf

# Zurück zu Standard:
SMB = Off
HTTP = Off
```

### Verification


```bash
# Config prüfen
cat /etc/responder/Responder.conf | grep -E "SMB|HTTP"
```

---

## ProxyChains Configuration zurücksetzen

### Standard Config wiederherstellen


```bash
# ProxyChains Config prüfen
cat /etc/proxychains4.conf

# Falls modifiziert, SOCKS5 Einträge entfernen
sudo nano /etc/proxychains4.conf
# Eigene Einträge löschen
```

---

## SMB Server beenden

### smbserver.py stoppen


```bash
# SMB Server Prozess finden
ps aux | grep smbserver

# Prozess beenden
pkill -f smbserver.py
```

---

## Temporary Files löschen

### Lokale Hash Files


```bash
# Hash Files entfernen
rm hashes.txt
rm ntlm_hashes.txt
rm kerb_hashes.txt
rm hash.txt
rm keepass.hash
```

### NTDS.dit Dumps


```bash
# Domain Dumps löschen
rm domain_hashes.txt
rm parent_domain.txt
```

### BloodHound JSON Files


```bash
# BloodHound Collection löschen
cd bloodhound-analysis
rm *.json
cd ..
```

### mitm6 Loot Directory


```bash
# mitm6 gesammelte Daten löschen
rm -rf lootme/
```

---

## ZeroLogon Restoration (wenn ausgeführt)

### Domain Controller Password wiederherstellen


```bash
# Step 1: Original Machine Account Hash aus secretsdump
# Format: ODIN-DC$:aad3b435b51404eeaad3b435b51404ee:ORIGINAL_HASH

# Step 2: Password restoration
python3 restorepassword.py MARVEL.local/ODIN-DC@ODIN-DC -target-ip 192.168.xx.x -hexpass [EXTRACTED_HEX_PASSWORD]
```

### Verification


```bash
# DC Funktionalität testen
crackmapexec smb 192.168.xx.x -u 'ODIN-DC$' -H [ORIGINAL_HASH]
```

---

## Impacket Installation zurücksetzen

### Modified Impacket entfernen (nach PrintNightmare)


```bash
# Modified Impacket deinstallieren
pip3 uninstall impacket

# Standard Impacket neu installieren
sudo apt install impacket-scripts
```

---

## NFS Mount Cleanup

### NFS Share unmounten


```bash
# NFS Share unmounten
sudo umount /tmp/mount

# Mount Point entfernen
sudo rm -rf /tmp/mount
```

---

## Database Cleanup

### CrackMapExec Database leeren


```bash
# CME Database öffnen
cmedb
```


```sql
# Alle Hosts löschen
DELETE FROM hosts;

# Alle Credentials löschen
DELETE FROM credentials;

# Exit
exit
```

---

## Hosts File zurücksetzen

### DNS Einträge entfernen


```bash
# /etc/hosts editieren
sudo nano /etc/hosts

# Folgende Zeilen entfernen:
# 10.10.175.182 mail01.hybrid.vl
# 10.10.175.181 dc01.hybrid.vl
# 10.10.255.70 lab.trusted.vl labdc.lab.trusted.vl
# 10.10.255.69 trusted.vl trusteddc.trusted.vl
```

---

## Neo4j / BloodHound Cleanup

### BloodHound Data löschen


```bash
# In BloodHound GUI:
# Settings -> Clear Database

# Oder via Cypher:
# MATCH (n) DETACH DELETE n
```

### Neo4j beenden


```bash
# Neo4j Service stoppen
sudo systemctl stop neo4j

# Neo4j Status prüfen
sudo systemctl status neo4j
```

---

## Working Directories aufräumen

### Domain-spezifische Directories


```bash
# Working Directory löschen
rm -rf marvel.local/
rm -rf bloodhound-analysis/
```

### Tool Repositories


```bash
# CVE Exploit Directories
rm -rf CVE-2021-1675/
rm -rf CVE-2020-1472/

# PlumHound Reports
rm -rf /opt/PlumHound/reports/
```

---

## OPSEC Considerations

### Forensische Spuren minimieren

**Services nicht erstellt:**

- wmiexec.py (verwendet WMI statt Service)
- smbexec.py (File-based statt Service)

**Services erstellt (hinterlässt Logs):**

- psexec.py (erstellt Windows Service)
- Metasploit PSExec (erstellt Service)

### Log-intensive Aktionen

**Hinterlassen viele Logs:**

- Domain User Creation
- Group Membership Changes
- Multiple Failed Auth Attempts
- PSExec Service Creation
- RDP Connections

**Weniger Logs:**

- WMIExec
- Pass-the-Hash
- Token Impersonation
- Golden Tickets (keine echten Logins)

---

## Cleanup Verification Checklist

### Domain Environment

- ✅ Alle erstellten Domain Users gelöscht
- ✅ Hochgeladene Tools entfernt
- ✅ LNK Files gelöscht
- ✅ Webshells entfernt
- ✅ ZeroLogon DC restored (falls ausgeführt)

### Local Environment

- ✅ Alle Shell Sessions beendet
- ✅ SSH Tunnels geschlossen
- ✅ SSHuttle gestoppt
- ✅ Responder beendet
- ✅ mitm6 / ntlmrelayx gestoppt
- ✅ SMB Server beendet
- ✅ Metasploit Sessions geschlossen

### Configuration Files

- ✅ Responder Config zurückgesetzt
- ✅ ProxyChains Config bereinigt
- ✅ /etc/hosts Einträge entfernt
- ✅ Modified Impacket entfernt (falls installiert)

### Data Files

- ✅ Hash Files gelöscht
- ✅ NTDS.dit Dumps gelöscht
- ✅ BloodHound JSON Files gelöscht
- ✅ mitm6 Loot Directory gelöscht
- ✅ Working Directories aufgeräumt
- ✅ CME Database geleert

### Processes

- ✅ Keine Responder Prozesse
- ✅ Keine mitm6 Prozesse
- ✅ Keine ntlmrelayx Prozesse
- ✅ Keine SSH Tunnels
- ✅ Keine SSHuttle Prozesse
- ✅ Keine SMB Server Prozesse

---

## Complete Cleanup Workflow

### Phase 1: Domain Cleanup (0-15 min)

cmd

```cmd
# 1. Backdoor Users entfernen
net user backdoor /delete /domain
net user hawkeye /delete /domain
net user harry /delete /domain

# 2. Tools auf Targets löschen
del C:\Windows\Temp\mimikatz.exe
del C:\Windows\Temp\*.sys
del C:\Windows\Temp\*.dll

# 3. LNK Files entfernen
del C:\important_document.lnk

# 4. Webshells entfernen
del C:\xampp\htdocs\dev\back.php
```

### Phase 2: Process Cleanup


```bash
# 1. Alle Attack Processes beenden
sudo pkill responder
sudo pkill mitm6
pkill -f ntlmrelayx
pkill -f smbserver.py

# 2. Tunnels schließen
pkill -f "ssh -f -N -D"
sudo pkill sshuttle

# 3. Shells beenden
# (exit in allen offenen Sessions)

# 4. Metasploit cleanup
# sessions -K in msfconsole
```

### Phase 3: Configuration Cleanup


```bash
# 1. Responder Config
sudo nano /etc/responder/Responder.conf
# SMB = Off, HTTP = Off

# 2. Hosts File
sudo nano /etc/hosts
# Target Einträge entfernen

# 3. ProxyChains Config prüfen
cat /etc/proxychains4.conf
# Falls modifiziert, zurücksetzen

# 4. Modified Impacket entfernen (falls installiert)
pip3 uninstall impacket
sudo apt install impacket-scripts
```

### Phase 4: Data Cleanup


```bash
# 1. Hash Files löschen
rm hashes.txt ntlm_hashes.txt kerb_hashes.txt

# 2. NTDS.dit Dumps
rm domain_hashes.txt parent_domain.txt

# 3. BloodHound Data
rm bloodhound-analysis/*.json

# 4. mitm6 Loot
rm -rf lootme/

# 5. Working Directories
rm -rf marvel.local/ hybrid.vl/

# 6. CME Database
cmedb
# DELETE FROM hosts; DELETE FROM credentials; exit
```

### Phase 5: Verification


```bash
# 1. Process Check
ps aux | grep responder
ps aux | grep mitm6
ps aux | grep ssh
ps aux | grep sshuttle
netstat -tlnp | grep 9050

# 2. Config Check
cat /etc/responder/Responder.conf | grep -E "SMB|HTTP"
cat /etc/hosts | grep -E "marvel|hybrid|trusted"

# 3. File Check
ls -la *.txt
ls -la *.json
ls -la lootme/ 2>/dev/null

# 4. Domain Check (via DC Access)
net user /domain
# Verify keine Backdoor Users vorhanden
```

---

## Troubleshooting

### Processes lassen sich nicht beenden


```bash
# Force Kill mit SIGKILL
kill -9 [PID]

# Alle Responder Instanzen
sudo pkill -9 -f responder

# Root Prozesse
sudo kill -9 [PID]
```

### SSH Tunnel hängt


```bash
# SSH Verbindungen auflisten
ss -tp | grep ssh

# Force Close
sudo pkill -9 -f "ssh -f -N -D"
```

### NFS Mount lässt sich nicht unmounten


```bash
# Force Unmount
sudo umount -f /tmp/mount

# Lazy Unmount
sudo umount -l /tmp/mount
```

### Domain User Deletion Failed

cmd

```cmd
# Prüfen ob User noch eingeloggt
query user /server:DC01

# User logoff forcen
logoff [SESSION_ID] /server:DC01

# Erneut löschen
net user hawkeye /delete /domain
```

---

## Post-Cleanup Verification

### System Status

- ✅ Keine laufenden Attack Prozesse
- ✅ Keine offenen SSH Tunnels
- ✅ Keine Listening SOCKS Ports
- ✅ Responder/mitm6 beendet

### Domain Status

- ✅ Keine Backdoor Accounts vorhanden
- ✅ Tools von Targets entfernt
- ✅ Malicious Files gelöscht
- ✅ Domain Groups unverändert

### Local System

- ✅ Configs zurückgesetzt
- ✅ Hosts File bereinigt
- ✅ Hash Files gelöscht
- ✅ Working Directories aufgeräumt
