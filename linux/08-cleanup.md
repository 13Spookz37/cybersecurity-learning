# Cleanup

## 1. Ziel der Cleanup Phase

- Spuren des Pentests entfernen
- System in ursprünglichen Zustand zurückversetzen
- sicherstellen das keine Backdoors oder Modifikationen zurückbleiben.

---

## 2. Relevante Techniken & Vorgehensweisen

### Artifact Removal

- Hochgeladene Tools entfernen
- Temporäre Files löschen
- Modifizierte Scripts zurücksetzen

### Configuration Restoration

- PAM Configs zurücksetzen
- /etc/passwd Modifikationen entfernen
- Systemdateien wiederherstellen

### Backdoor Removal

- SSH Keys entfernen
- Backdoor User löschen
- PAM Backdoors deaktivieren

### File Cleanup

- Test-Files löschen
- Exploit-Binaries entfernen
- Downloaded Tools entfernen

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- `/tmp` Directory als primärer Upload-Ort
- `/etc/passwd` für User-Modifikationen
- `/etc/pam.d/` für Auth-Modifikationen
- `~/.ssh/authorized_keys` für SSH-Backdoors
- Shared Mounts zwischen Container/Host

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### Temporäre Files

- Tools in `/tmp`
- Scripts in `/tmp`
- Exploit Binaries

### System Configuration

- Modified `/etc/passwd`
- Modified `/etc/pam.d/sshd`
- Planted SSH Keys

### Test Artifacts

- Test Directories
- Test Files aus Container Escape
- SUID Binaries für Exploitation

---

## 5. Wichtige Befehle, Tools & Syntax

### Uploaded Tools entfernen

**Enumeration Tools**


```bash
# LinPEAS
rm /tmp/linpeas.sh

# pspy
rm /tmp/pspy64

# Custom Scripts
rm /tmp/backdoor.sh
rm /tmp/exploit.sh
```

**Exploit Binaries**


```bash
# PHP Reverse Shell
rm /var/www/html/survey/upload/plugins/Y1LD1R1M/php-rev.php

# LimeSurvey Plugin
rm -rf /var/www/html/survey/upload/plugins/Y1LD1R1M/

# Uploaded Shells
rm /var/www/html/shell.php
rm shell.elf
```

**Container Escape Artifacts**


```bash
# SUID Binary (Container)
rm /var/www/html/survey/pwn

# SUID Binary (Host)
rm /opt/limesurvey/pwn

# Test Directories
rm -rf /var/www/html/survey/test-from-container
rm -rf /opt/limesurvey/test-from-container
```

**NFS SUID Binaries**


```bash
# Auf Host
rm /home/bob/bash
```

### SSH Keys entfernen

**authorized_keys löschen**


```bash
# Für alice
rm /home/alice/.ssh/authorized_keys

# Für root/root2
rm /root/.ssh/authorized_keys

# Alternativ: Nur spezifischen Key entfernen
nano ~/.ssh/authorized_keys
# Zeile mit eigenem Public Key löschen
```

### Backdoor User entfernen

**User aus /etc/passwd entfernen**


```bash
# Mit sed
sed -i '/^root2:/d' /etc/passwd

# Manuell
nano /etc/passwd
# Zeile mit root2: löschen
```

**Vorher prüfen:**


```bash
grep root2 /etc/passwd
```

### PAM Configuration zurücksetzen

**Backdoor Entry entfernen**


```bash
nano /etc/pam.d/sshd
# Letzte Zeile entfernen:
# auth required pam_exec.so /tmp/backdoor.sh
```

**Backdoor Script entfernen**


```bash
rm /tmp/backdoor.sh
```

### Modified Scripts zurücksetzen

**Backup wiederherstellen**


```bash
# Wenn Backup existiert
mv script.sh.bak script.sh

# Original Script von Backup
cp backup/original_script.sh /path/to/script.sh
```

**Cron Script Restore**


```bash
# marcy_script.sh
mv /home/alice/marcy_script.sh.bak /home/alice/marcy_script.sh
```

### Unmount NFS/SMB Shares

**NFS Unmount (auf Kali)**


```bash
sudo umount mount
```

**SMB Unmount (auf Kali)**


```bash
sudo umount mount
```

### Container Cleanup

**Test Files entfernen**


```bash
# Im Container
rm -rf /var/www/html/survey/test-from-container

# Auf Host
rm -rf /opt/limesurvey/test-from-container
```

**SUID Binaries**


```bash
# Container
rm /var/www/html/survey/pwn

# Host
rm /opt/limesurvey/pwn
```

### Database Cleanup (optional)

**MariaDB auf Kali stoppen**


```bash
sudo systemctl stop mariadb
```

**Database löschen (falls angelegt)**


```bash
sudo mariadb -u root
```


```sql
DROP DATABASE limesurveydb;
DROP USER 'limesurveyuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

### File Permission Restoration

**Falls Permissions geändert wurden**


```bash
# Original Permissions wiederherstellen
chmod 644 /path/to/file
chmod 755 /path/to/directory
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Cleanup Workflow

1. **Backdoors entfernen (PRIORITÄT)**


```bash
   # SSH Keys
   rm ~/.ssh/authorized_keys
   
   # PAM Backdoor
   nano /etc/pam.d/sshd  # Entry entfernen
   rm /tmp/backdoor.sh
   
   # Backdoor User
   sed -i '/^root2:/d' /etc/passwd
```

2. **Uploaded Tools löschen**


```bash
   # /tmp aufräumen
   rm /tmp/linpeas.sh
   rm /tmp/pspy64
   rm /tmp/*.sh
```

3. **Exploit Artifacts entfernen**


```bash
   # Web Shells
   rm /var/www/html/shell.php
   
   # Plugins
   rm -rf /var/www/html/survey/upload/plugins/Y1LD1R1M/
   
   # SUID Binaries
   rm /opt/limesurvey/pwn
```

4. **Modified Scripts zurücksetzen**


```bash
   mv script.sh.bak script.sh
```

5. **Test Files entfernen**


```bash
   rm -rf /path/to/test-directory
```

6. **Mounts trennen (auf Kali)**


```bash
   sudo umount mount
```

7. **Verification**


```bash
   # Prüfen ob sauber
   ls /tmp/
   grep root2 /etc/passwd
   cat /etc/pam.d/sshd
   ls ~/.ssh/
```

### Wichtige Hinweise

**Snapshot VOR destructiven Actions**

- CP SUID Exploitation kann /etc/passwd brechen
- Immer Snapshot machen
- Bei echten Pentests: Backup vor Änderungen

**PAM Backdoor Cleanup KRITISCH**


```bash
# Wenn nicht entfernt: User können sich nicht einloggen
# IMMER cleanup durchführen
nano /etc/pam.d/sshd
# auth required pam_exec.so /tmp/backdoor.sh  ← LÖSCHEN
```

**SSH Keys Cleanup**


```bash
# Nur eigene Keys entfernen
# NICHT alle Keys löschen (bricht legitime Logins)
nano ~/.ssh/authorized_keys
# Nur eigene Zeile löschen
```

**Backdoor User Check**


```bash
# Vor Entfernen prüfen
grep root2 /etc/passwd

# Nach Entfernen verifizieren
grep root2 /etc/passwd
# Sollte leer sein
```

**/tmp ist world-writable**


```bash
# Andere User-Files nicht löschen
ls -la /tmp/
# Nur eigene Files entfernen
```

**Container vs Host Cleanup**

- Files im Container bleiben nicht nach Container-Stop
- Files auf Host-Mounts bleiben persistent
- Beide Locations prüfen

**NFS/SMB Mounts**


```bash
# Auf Kali unmounten
sudo umount mount

# Mount Point bleibt leer zurück (OK)
ls mount/
# Sollte leer sein
```

**Modified Script Restore**


```bash
# Backup vorhanden?
ls script.sh.bak

# Restore
mv script.sh.bak script.sh

# Keine Backup? Original-Content aus Notes rekonstruieren
```

**Permissions nach Cleanup**


```bash
# Original Permissions wiederherstellen
# Beispiel:
chmod 755 script.sh
chown user:user script.sh
```

**Database Cleanup bei LimeSurvey**


```bash
# MariaDB auf Kali läuft noch
# Optional: Stoppen
sudo systemctl stop mariadb

# Optional: Database entfernen
# Nur in Lab-Umgebung
```

**Verification Commands**


```bash
# /tmp prüfen
ls -la /tmp/ | grep -E "linpeas|pspy|exploit"

# /etc/passwd prüfen
grep -E "root2|backdoor|hacker" /etc/passwd

# PAM prüfen
grep pam_exec /etc/pam.d/sshd

# SSH Keys prüfen
cat ~/.ssh/authorized_keys | grep kali
```

---

## 7. Typische Findings & Fehler

### Häufige Findings

**Backdoors noch vorhanden**


```bash
# Nach Cleanup prüfen
cat ~/.ssh/authorized_keys
# Sollte leer oder nur legitime Keys enthalten

grep root2 /etc/passwd
# Sollte keine Zeile zurückgeben

grep pam_exec /etc/pam.d/sshd
# Sollte keine Zeile zurückgeben
```

**Tools in /tmp vergessen**


```bash
ls /tmp/
# linpeas.sh, pspy64 noch vorhanden
```

**SUID Binaries auf Host**


```bash
ls -la /opt/limesurvey/pwn
# -rwsr-xr-x  ← Noch vorhanden
```

**Test Directories**


```bash
ls -la /opt/limesurvey/
# test-from-container/ noch vorhanden
```

### Typische Fehler

**PAM Backdoor nicht entfernt**


```bash
# User versucht SSH Login
ssh alice@target
# Hängt

# Grund: PAM Backdoor noch aktiv
# Fix:
nano /etc/pam.d/sshd
# Zeile entfernen
```

**Backdoor User: Falsche Zeile gelöscht**


```bash
# Vor Cleanup
cat /etc/passwd
# root:x:0:0:...
# root2:hash:0:0:...

# Fehler: root statt root2 gelöscht
sed -i '/^root:/d' /etc/passwd

# System kaputt!
# Fix: Snapshot restore
```

**SSH Keys: Alle Keys gelöscht**


```bash
# Falsch
rm ~/.ssh/authorized_keys

# Problem: Legitime User können sich nicht einloggen
# Richtig: Nur eigene Zeile aus authorized_keys entfernen
nano ~/.ssh/authorized_keys
```

**Modified Script: Backup überschrieben**


```bash
# Falsch
mv script.sh script.sh.bak  # Überschreibt existierendes Backup

# Richtig: Backup prüfen
ls script.sh.bak
# Wenn vorhanden, NICHT überschreiben
```

**Container Files vs Host Files**


```bash
# Im Container gelöscht
rm /var/www/html/survey/pwn

# Aber auf Host noch vorhanden
ls /opt/limesurvey/pwn
# Datei existiert noch

# Fix: Beide Locations cleanen
```

**Permissions falsch gesetzt**


```bash
# Nach Cleanup
ls -la script.sh
# -rw------- (nur Owner kann lesen)

# Original war
# -rwxr-xr-x (executable für alle)

# Fix:
chmod 755 script.sh
```

**Mounts nicht getrennt**


```bash
# Mount noch aktiv
mount | grep mount
# TARGET:/share on /home/kali/mount

# Problem: Zugriff auf Target Files
# Fix:
sudo umount mount
```

**Tools nicht aus allen Locations entfernt**


```bash
# /tmp gecleant
ls /tmp/linpeas.sh
# Nicht gefunden

# Aber in /home/user noch vorhanden
ls ~/linpeas.sh
# Datei existiert

# Fix: Alle Upload-Locations prüfen
```

**Database Connection noch offen**


```bash
# MariaDB auf Kali läuft noch
sudo systemctl status mariadb
# active (running)

# Target könnte noch connecten
# Best Practice: Stoppen nach Test
sudo systemctl stop mariadb
```

**Reverse Shell Payloads vergessen**


```bash
# Web Shell noch im DocumentRoot
ls /var/www/html/shell.php
# Datei existiert

# Plugins noch vorhanden
ls /var/www/html/survey/upload/plugins/Y1LD1R1M/
# Directory existiert

# Fix: Alle Web Shells entfernen
```
