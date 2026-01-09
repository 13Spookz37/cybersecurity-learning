# Lab: Build

**Plattform:** VulnLab  
**Kategorie:** Maschine  
**Betriebssystem:** Linux  
**Schwierigkeit:** Easy  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Build simuliert eine Entwicklungs und Build-Umgebung, wie sie in vielen realen
Unternehmensnetzen existiert.

Der Fokus liegt nicht auf einem einzelnen verwundbaren Service, sondern auf
Zusammenspiel, Vertrauen und Annahmen innerhalb interner Systeme.

Ziel war es zu verstehen:
- wie Build und Entwicklungsumgebungen Angriffsflächen erzeugen
- warum interne Dienste oft unterschätzt werden
- wie Designentscheidungen wichtiger sein können als klassische Exploits

---

## 🧠 Methodischer Ansatz

Der Einstieg begann ohne klare Richtung.
Mehrere erreichbare Services, mehrere mögliche Pfade.

Der erste Impuls war klassisch:
- Web
- offensichtlicher Service
- direkter technischer Einstieg

CI/CD-nahe Komponenten habe ich zunächst eher als Kontext wahrgenommen,
nicht als Kern des Angriffs.

Rückblickend war genau das der Denkfehler.

---

## 🔍 Reconnaissance

### Beobachtungen

- Mehrere Dienste existieren parallel
- Interne Services sind erreichbar
- Backups sind zugänglich
- Build-Artefakte enthalten Informationen, die nicht öffentlich sein sollten

Keiner dieser Punkte war für sich genommen kritisch.
Auffällig wurde es erst im Zusammenhang.

### Schlussfolgerungen

- Der Einstieg liegt nicht in einem einzelnen Dienst
- Die Angriffsfläche entsteht durch Verknüpfung
- Vertrauen zwischen Komponenten ist der eigentliche Hebel

---

## 🚪 Initial Access

Der initiale Zugriff erfolgte nicht über einen klassischen Exploit,
sondern über einen vorgesehenen, aber falsch gedachten Zugriffspfad.

Nicht „kaputt“, sondern **zu offen**.

Der entscheidende Punkt war nicht Technik,
sondern das implizite Vertrauen innerhalb der Umgebung.

---

## 🔼 Privilege Escalation

Nach dem initialen Zugriff war klar:
Die Umgebung vertraut internen Prozessen mehr als externen Nutzern.

Dieses Vertrauen ließ sich ausweiten.
Nicht durch Komplexität, sondern durch Logik.

Die Eskalation ergab sich aus dem Design,
nicht aus einer einzelnen Schwachstelle.

---

## 🏁 Ergebnis

- Vollständiger Zugriff auf das Zielsystem
- Kontrolle über Build-nahe Prozesse
- Missbrauch interner Vertrauensbeziehungen

Die Maschine fiel nicht wegen eines Exploits,
sondern wegen Annahmen.

---

## 📚 Learnings & Reflexion

### Technisch

- Interne Dienste sind oft hochprivilegiert
- Backups sind reale Einstiegspunkte
- Build und Dev-Umgebungen brauchen klare Grenzen

### Methodisch

- Klassisches Exploit Denken kann in die falsche Richtung führen
- Verbindungen sind wichtiger als einzelne Services
- Designfehler schlagen Schwachstellen

### Rückblickend

- Zu lange nach einem „Bug“ gesucht
- Zu spät akzeptiert, dass der Angriffspfad logisch ist
- Nächstes Mal: früher hinterfragen, was als „intern“ gilt

---

👉 **Technischer Walkthrough**: [Medium](https://medium.com/@13spookz37/build-vm-walkthrough-b28c89d45c63)

---

> **Hinweis:**  
> Dieses Write-Up beschreibt den Denk und Entscheidungsprozess.  
> Es enthält bewusst keine Spoiler oder reproduzierbaren Schritte.
