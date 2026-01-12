# Lab: CloudGoat AWS Beanstalk Secrets

**Plattform:** CloudGoat  
**Schwierigkeit:** Easy  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · IAM · Elastic Beanstalk · STS · Environment Variables  
**Status:** Completed


---

## 🎯 Ziel des Labs

Dieses Lab beschäftigt sich mit Sicherheitsrisiken, die nicht durch Exploits, sondern durch **Architektur und Konfigurationsentscheidungen** entstehen.

Im Fokus stand nicht:

- ein klassischer Bug
- ein Service‑Exploit
- eine ungewöhnliche Angriffstechnik

Sondern das Verständnis dafür,

- wie PaaS‑Konfigurationen selbst zur Angriffsfläche werden
- warum Leserechte in Cloud‑Umgebungen selten harmlos sind
- wie mehrere legitime Entscheidungen in Kombination zu vollständiger Eskalation führen können

Das primäre Lernziel war zu verstehen, dass **Credentials oft an unerwarteten Stellen liegen** und dass **IAM‑Rechte erst im Zusammenspiel ihre tatsächliche Wirkung entfalten**.

---

## 🧠 Methodischer Ansatz

Meine erste Annahme war eine typische Cloud‑Fehlkonfiguration, die sich direkt ausnutzen lässt.

Diese Erwartung erwies sich schnell als zu kurz gedacht.

Statt nach Exploits zu suchen, habe ich den Fokus bewusst auf:

- das Lesen von Berechtigungen im Kontext
- das Hinterfragen von Service‑Zugriffen
- das Verständnis von Informationsflüssen zwischen AWS‑Komponenten

gelegt.

Der entscheidende Schritt war, **nicht nach dem nächsten Angriff**, sondern nach **implizitem Vertrauen und Datenexposition** zu suchen.

> ⚠️ Fokus auf Denkprozess und Architekturverständnis.

---

## 🔍 Reconnaissance

### Beobachtungen

- Der initiale IAM‑User verfügte nur über eingeschränkte Rechte
- Keine IAM‑Schreibrechte
- Keine offensichtlichen Eskalationsmöglichkeiten
- Elastic Beanstalk war der einzige Service mit relevantem Zugriff

Auffällig war dabei weniger **was erlaubt war**, sondern **wo Leserechte existierten**:

- Anwendungen und Environments waren vollständig einsehbar
- Konfigurationswerte konnten uneingeschränkt gelesen werden

### Schlussfolgerungen

Daraus ergaben sich keine direkten Exploits, aber mehrere Hypothesen:

- Konfigurationsdaten könnten sensible Informationen enthalten
- Beanstalk dient nicht nur dem Deployment, sondern auch als Konfigurationsspeicher
- „Read‑Only“ auf PaaS‑Services kann faktisch Credential‑Zugriff bedeuten

Bewusst ausgeschlossen wurden:

- direkte IAM‑Manipulation
- service‑spezifische Exploits
- brute‑force‑artige Ansätze

---

## 🚪 Initial Access

### Einstiegspunkt

Der initiale Zugriff war bereits vorhanden:  
ein Low‑Privileged AWS‑User mit gültigen Credentials.

Der eigentliche Erkenntnisgewinn lag nicht im Zugriff selbst, sondern in der Frage:

- **Welche Informationen darf diese Identität sehen?**
- **Wie können diese Informationen weiterverwendet werden?**

Der Einstiegspunkt war damit nicht ein technischer Fehler, sondern **offen zugängliche Konfiguration**.

### Wichtige Erkenntnisse

- Environment Variables sind kein sicherer Ort für Secrets
- Leserechte auf Konfiguration bedeuten oft faktischen Zugriff auf Credentials
- Die Annahme, dass Secrets ausschließlich vom Anwendungscode genutzt werden, ist eindeutig eine Fehlannahme.

Der Fehler lag nicht in der Technik, sondern im **Design‑Verständnis**.

---

## 🔼 Privilege Escalation

### Analyse

Nach dem Wechsel des Users zeigte sich:

- keine direkten Administratorrechte
- umfangreiche IAM‑Lesezugriffe
- eine einzelne, hochkritische Berechtigung: `iam:CreateAccessKey` auf `*`

Isoliert betrachtet wirkt diese Permission harmlos – im Kontext ist sie entscheidend.

### Entscheidungsfindung

Der gewählte Eskalationspfad basierte auf Logik, nicht auf Kreativität:

- keine Trust‑Beziehungen erforderlich
- keine Rollenannahme notwendig
- keine Manipulation der Infrastruktur

Die Eskalation entstand ausschließlich durch die **legitime Nutzung vorhandener Rechte**.

Andere Wege wurden verworfen, da sie:

- komplexer
- auffälliger
- oder abhängig von weiteren Fehlkonfigurationen gewesen wären

Zentrale Erkenntnis:
**Legitime Aktionen sind oft gefährlicher als Exploits.**

---

## 🏁 Ergebnis

- Vollständiger administrativer Zugriff auf den AWS‑Account
- Zugriff auf Secrets Manager und weitere sensitive Ressourcen
- Persistenz durch zusätzliche Access Keys möglich

Die Angriffskette bestand nicht aus einem einzelnen Schritt, sondern aus einer logisch konsistenten Abfolge:

**Low-Priv User**
→ Leserechte auf Beanstalk-Konfiguration
→ Offenliegende Credentials
→ IAM-Analyse
→ Legitimes Erstellen neuer Access Keys
→ Vollzugriff

---

## 📚 Learnings & Reflexion

### Was habe ich gelernt?

**Technisch**
- Environment Variables sind ein häufig unterschätztes Risiko
- `iam:CreateAccessKey` gehört zu den gefährlichsten IAM‑Rechten
- PaaS‑Services werden in Security‑Audits oft nicht tief genug betrachtet

**Methodisch**
- Berechtigungen müssen im Zusammenspiel gelesen werden
- „Read‑Only“ ist kein Sicherheitskonzept
- Jede Identität erfordert eine eigene Analyse

**Gedanklich**
- Information Disclosure ist häufig der eigentliche Einstiegspunkt
- Entwickler‑Convenience kollidiert regelmäßig mit Security
- Sicherheitsprobleme entstehen oft durch legitime Designentscheidungen

### Was würde ich nächstes Mal anders machen?

- Früher hinterfragen, warum bestimmte Zugriffe existieren
- Konfigurationsdaten schneller als Angriffsfläche erkennen
- mehr Analyse der AWS‑Architektur

---

👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-beanstalk-secrets-walkthrough-aee3c92d9d29)**

---

> 💡 **Hinweis:**  
> Dieses Write‑Up dokumentiert Denk‑ und Entscheidungsprozesse.  
> Es enthält bewusst **keine reproduzierbaren Schritt‑für‑Schritt‑Anleitungen**.
