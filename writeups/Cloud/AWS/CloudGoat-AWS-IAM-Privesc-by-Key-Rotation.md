# Lab: CloudGoat AWS IAM Privilege Escalation by Key Rotation

**Plattform:** CloudGoat  
**Schwierigkeit:** Easy  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · IAM · MFA · Key Rotation  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Im Mittelpunkt stand eine Sicherheitsmaßnahme, die eigentlich schützen soll:
**Access Key Rotation** – ergänzt durch MFA, die interaktive Logins absichert.

Das Ziel war zu verstehen,
- wie gut gemeinte Sicherheitsprozesse neue Angriffsflächen erzeugen können
- warum legitime IAM‑Funktionen ausreichen, um die Kontrolle zu verlieren
- und weshalb man Berechtigungen nicht isoliert, sondern im Ablauf betrachten muss

Ich habe gelernt, dass:
**Key Rotation nicht automatisch als Schutz zu sehen, sondern als Prozess mit Risiko wenn man es falsch implementiert**.

---

## 🧠 Methodischer Ansatz

Ein zentraler Punkt in diesem Lab war die Annahme,
dass Key Rotation in Kombination mit MFA automatisch Sicherheit schafft.

MFA war aktiviert.
Rotation war vorgesehen.
Und trotzdem reichte ein einzelner, gültiger Access Key aus,
um die bestehende Sicherheitslogik zu umgehen.

Nicht, weil MFA „umgangen“ oder „gebrochen“ wurde,
sondern weil sie für diesen Zugriff schlicht keine Rolle spielte.

MFA schützt interaktive Logins.
Access Keys sind davon entkoppelt.

Der Fehler lag nicht im Mechanismus,
sondern in der Erwartung,
dass MFA auch dort wirkt,
wo ausschließlich über programmatische Schlüssel gearbeitet wird.


---

## 🔍 Reconnaissance

### Beobachtungen

Der initiale IAM‑User war eingeschränkt:
- keine administrativen Rechte
- keine Möglichkeit, Policies zu verändern
- kein direkter Zugriff auf sensitive Services

Auffällig war jedoch:
- Berechtigungen rund um Access Keys
- die Möglichkeit, neue Schlüssel zu erzeugen
- und bestehende Schlüssel zu rotieren

Das war auch der initiale Escalations Vektor.

### Schlussfolgerungen

Die Key Rotation wurde hier als reine Schutzmaßnahme verstanden:
- alte Schlüssel raus
- neue Schlüssel rein
- Sicherheit erhöht

Was dabei übersehen wurde:
**Wer neue Schlüssel erzeugen darf, hat die volle Kontrolle über die Identitäten.**

---

## 🚪 Initial Access

Der Zugriff war von Anfang an gegeben:
ein Low‑Privileged IAM‑User mit gültigen Credentials.

Der eigentliche Einstiegspunkt war kein Fehler,
sondern eine erlaubte Handlung:
**das Erstellen neuer Access Keys**.

In diesem Moment hat sich der Zugriff nicht „erhöht“,
aber **er hat sich vervielfacht**.

Ein zusätzlicher Schlüssel bedeutet:
- mehr Angriffsfläche
- mehr Persistenz
- mehr Möglichkeiten zur unbemerkten Nutzung

Kein Hack.
Kein Bruch.
Nur AWS‑Standardverhalten.

---

## 🔼 Privilege Escalation

### Analyse

Es gab hier keine klassische Eskalation im Sinne von:
User → Admin.

Stattdessen:
- bestehende Rechte
- legitime Schlüsselrotation
- neue, vollwertige Access Keys

Die Eskalation lag nicht in höheren Rechten,
sondern in der **Kontrolle über Zugangsdaten**.

Wer Schlüssel erzeugen darf,
kontrolliert den Zugang –
auch ohne zusätzliche Permissions.

### Entscheidungsfindung

In diesem Lab ging es weniger darum, neue Rechte zu bekommen,  
sondern zu erkennen, **wie bestehende Prozesse unbemerkt zur Kontrolle führen können**.

Key Rotation wurde intuitiv als Schutzmaßnahme gesehen –  
aber genau diese Annahme öffnete die Tür.  
Es war weniger ein „Angriff“, sondern eher das Verstehen einer Schwachstelle im Design.

---

## 🏁 Ergebnis

- Persistenter Zugriff über neu erzeugte Access Keys
- Umgehung der ursprünglichen Zugriffsbeschränkung
- Kontrolle über IAM‑Zugänge ohne Policy‑Änderung

Die Kette war unspektakulär, aber deswegen nicht weniger gefährlich:

Low‑Priv User  
→ Berechtigung zur Key Rotation  
→ Neue Access Keys  
→ Dauerhafter Zugriff  

Kein Exploit.
Kein Alarm.
Kein sichtbarer Bruch.

---

## 📚 Learnings & Reflexion

### Technisch
- Key Rotation ist kein Schutzmechanismus, sondern ein Prozess mit Risiko
- `iam:CreateAccessKey` ist eine der kritischsten IAM‑Berechtigungen
- MFA schützt keine programmatischen Zugriffe über Access Keys
- Neue Schlüssel sind neue Zugangspunkte – unabhängig von der ursprünglichen Rolle

### Methodisch
- Sicherheitsmechanismen müssen im Ablauf betrachtet werden, nicht isoliert
- „Best Practice“ ersetzt kein Bedrohungsmodell
- Legitime Funktionen können genauso gefährlich sein wie verbotene Aktionen

### Persönlich
- Ich habe unterschätzt, wie viel Kontrolle scheinbar kleine IAM-Rechte tatsächlich geben können.  
- Dieses Lab hat mein Verständnis von AWS Methodik und den Möglichkeiten der CLI deutlich geschärft.

---

👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-aws-key-rotation-afdd37114679)**  


---

> 💡 **Hinweis:**  
> Dieses Write‑up beschreibt meinen Denk‑ und Lernprozess.  
> Es verzichtet bewusst auf technische Schritt‑für‑Schritt‑Anleitungen.
