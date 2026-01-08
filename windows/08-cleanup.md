# Cleanup


## Ziel der Cleanup Phase

- Spuren der Kompromittierung minimieren
- Originale System-Zustände wiederherstellen wo möglich
- Artefakte entfernen die auf Angriff hinweisen
- System-Stabilität erhalten (insbesondere Production)

---

## Relevante Techniken & Vorgehensweisen

### File Backup vor Modification

**Service Binary Backup:**

cmd

```cmd
copy BootTime.exe BootTime.exe.bak
```

**Konzept:**

- Original-Datei sichern vor Ersetzung
- Ermöglicht Wiederherstellung
- Wichtig in Production-Umgebungen

### File Cleanup

**Temporäre Dateien entfernen:**


```bash
# GTFOBins ZIP
sudo rm $TF
```

**NFS Unmount:**


```bash
sudo umount /mnt/dev
```

**Konzept:**

- Temporäre Exploit-Artefakte entfernen
- Mount-Points aufräumen

### Metasploit Cleanup

**Artifact Cleanup-Versuch:**

meterpreter

```meterpreter
[*] Attempting to clean up files...
```

**Konzept:**

- Metasploit versucht automatisch Cleanup
- Nicht immer erfolgreich
- Manuelle Überprüfung nötig

### DC Password Restore (Zerologon)

**Nach Zerologon-Exploit:**


````bash
python3 restorepassword.py retro2.vl/BLN01\$@BLN01 -target-ip <TARGET_IP> -hexpass <PLAIN_PASSWORD_HEX>
```

**LSA Secrets für Wiederherstellung:**
- `plain_password_hex` aus secretsdump-Ausgabe

**Konzept:**
- DC Machine Account Password wiederherstellen
- Verhindert Domain-Beschädigung
- Kritisch in Production

**Hinweis aus Notes:**
- Bei VulnLab-Labs optional (VMs werden zurückgesetzt)
- In Production IMMER durchführen

---

## Betriebssystem-spezifische Aspekte (Windows)

### Windows Event Logs

**Auffällige Events:**
- Volume Shadow Copy Erstellung
- NTDS.dit Zugriffe
- Domain User Creation
- Certificate Requests
- Service Modifications

**Spezielle Erwähnung aus meinen Notes:**
- Domain User Creation: Auffällig in Event Logs
- Certificate Request: Geloggt
- Volume Shadow Copy: Generiert Event Logs

### File System

**Backup-Dateien:**
```
BootTime.exe.bak
```

**Temporäre Exploit-Dateien:**
```
C:\Temp\ntds.dit
C:\Temp\SYSTEM
C:\Users\Public\SetOpLock.exe
winPEASx64.exe
Wise.exe
```

**Shadow Copy Mount:**
```
z:\ (Volume Shadow Copy)
````

### Active Directory

**DC Machine Account:**

- Zerologon setzt Passwort auf leer
- Domain kann permanent beschädigt werden ohne Restore

---

## Typische Angriffsflächen

### Modified Files

**Service Binaries:**

- WiseBootAssistant: BootTime.exe ersetzt
- Backup: BootTime.exe.bak erstellt

**Webshells:**

- back.php in C:/xampp/htdocs/dev/
- shell.aspx in Webroot

**Uploaded Tools:**

- winPEASx64.exe
- SetOpLock.exe
- Wise.exe

### Temporary Artifacts

**Volume Shadow Copy:**

- z:\ Mount
- extract.dsh Script

**Downloaded Files:**

- NTDS.dit in C:\Temp
- SYSTEM in C:\Temp

**NFS Mounts:**

- /mnt/dev
- /tmp/mount

### Created Accounts

**Domain Users:**

- harry:haxxor@12345
- Zu Domain Admins hinzugefügt

**Modified Groups:**

- Local Administrators
- Domain Admins

### Modified Passwords

**Computer Accounts:**

- BANKING$: Passwort geändert
- MAIL01$: Keytab extrahiert

**User Accounts (Pass-the-Cert):**

- Administrator: Neues Passwort via Certificate

---

## Wichtige Befehle & Syntax

### File Backup

cmd

```cmd
copy <ORIGINAL> <ORIGINAL>.bak
```

### File Cleanup


```bash
sudo rm <FILE>
sudo umount <MOUNT_POINT>
```

### DC Password Restore


```bash
python3 restorepassword.py <DOMAIN>/<DC$>@<DC_NAME> -target-ip <IP> -hexpass <HEX_PASSWORD>
```

### Optional Cleanup (GTFOBins)


```bash
sudo rm $TF
```

---

## Reihenfolge & Abhängigkeiten

### Service Binary Modification Cleanup

**Best Practice:**

1. **Vor Modification** → Backup erstellen

cmd

```cmd
copy BootTime.exe BootTime.exe.bak
```

2. **Nach Exploit** → Original wiederherstellen (optional)

cmd

```cmd
copy BootTime.exe.bak BootTime.exe
```

3. **Backup entfernen** → Spuren minimieren

cmd

````cmd
del BootTime.exe.bak
```

**Beispiel (Butler VM):**
```
Vulnerable: C:\Program Files (x86)\Wise\Wise Care 365\BootTime.exe
Backup: copy BootTime.exe BootTime.exe.bak
Replace: copy Wise.exe BootTime.exe
Cleanup: [Nach Exploit optional]
````

### Volume Shadow Copy Cleanup

1. **Shadow Copy verwendet**
2. **Dateien heruntergeladen**
3. **Mount entfernen**

powershell

```powershell
# Shadow Copy automatisch removed nach diskshadow exit
```

4. **Temp-Dateien löschen**

cmd

```cmd
del C:\Temp\ntds.dit
del C:\Temp\SYSTEM
del extract.dsh
```

### Zerologon Cleanup Flow

1. **Zerologon ausgeführt** → DC Password = leer
2. **DCSync durchgeführt** → Hashes extrahiert
3. **LSA Secrets sichern** → plain_password_hex notieren
4. **DC Password restore**


```bash
python3 restorepassword.py retro2.vl/BLN01\$@BLN01 -target-ip <IP> -hexpass <HEX>
```

**Wichtigkeit:**

- **VulnLab:** Optional (VM-Reset)
- **Production:** KRITISCH (Domain-Beschädigung)

### NFS Cleanup

1. **NFS gemountet** → /mnt/dev, /tmp/mount
2. **Dateien kopiert**
3. **Unmount**


```bash
sudo umount /mnt/dev
```

### Uploaded Tool Cleanup

1. **Tools hochgeladen** → certutil, wget
2. **Nach Verwendung löschen**

cmd

````cmd
del C:\Users\Public\SetOpLock.exe
del winPEASx64.exe
del Wise.exe
```

---

## Typische Findings & Fehler

### Backup Best Practices

**Butler VM (Success):**
```
Original: BootTime.exe
Backup: copy BootTime.exe BootTime.exe.bak
Modified: copy Wise.exe BootTime.exe
Result: Original wiederherstellbar
```

**Hinweis:**
- "Always backup original files before replacing"

### Cleanup Attempts

**Metasploit (Limited Success):**
```
[*] Attempting to clean up files...
````

- Automatischer Cleanup-Versuch
- Nicht immer erfolgreich
- Manuelle Überprüfung nötig

**GTFOBins ZIP (Optional):**


````bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
# Optional Cleanup
sudo rm $TF
```

### Zerologon Cleanup

**Retro2 VM:**
```
Exploit: Zerologon (BLN01)
DC Password: Auf leer gesetzt
LSA Secrets: plain_password_hex gesichert
Restore: python3 restorepassword.py
Context: VulnLab (optional)
Production: KRITISCH
```

**Warnung:**
- "Ohne Wiederherstellung kann die Domain permanent beschädigt werden"

### NFS Unmount

**Dev VM:**
```
Mount: sudo mount -t nfs <IP>:/srv/nfs /mnt/dev
Usage: Dateien kopiert
Cleanup: sudo umount /mnt/dev
Result: Mount-Point entfernt
````

### Fehlende Cleanup-Aktionen

**Noch nicht erwähnt:**

- Event Log Clearing
- Webshell Removal nach Verwendung
- Created Domain User Deletion

**auffällig:**

- Domain User Creation: "Auffällig in Event Logs"
- Certificate Request: "Geloggt"
- Volume Shadow Copy: "Generiert Event Logs"

### OPSEC-Hinweise

**Domain User Creation:**

- Auffällig in Event Logs
- User-Name wählen der nicht auffällt
- Nicht zu "Domain Admins" hinzufügen wenn nicht nötig

**Hash Extraction:**

- Volume Shadow Copy generiert Event Logs
- NTDS.dit Download erkennbar
- Offline-Nutzung unerkennbar

**Certificate Persistence:**

- Certificate Request geloggt
- Wiederholte Authentication weniger auffällig als Login-Versuche

**File Modification:**

- Backup erstellen: "Always backup original files"
- Wichtig in Production: "Cleanup wichtig in Production"

### Production vs Lab Environment

**VulnLab-Labs:**

- VM-Reset möglich
- Zerologon Restore optional
- Cleanup weniger kritisch

**Production:**

- Zerologon Restore KRITISCH
- File Backups essentiell
- Cleanup wichtig für Stabilität
