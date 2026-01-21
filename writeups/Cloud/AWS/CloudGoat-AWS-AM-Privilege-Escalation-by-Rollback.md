# Lab: CloudGoat AWS IAM Privilege Escalation by Rollback

**Plattform:** CloudGoat  
**Schwierigkeit:** Easy  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · IAM · Policy Versioning  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Dieses Lab drehte sich um eine IAM‑Funktion, die im Alltag oft übersehen wird:
**Policy Versioning**.

Ziel war zu verstehen,
- warum alte Policy‑Versionen ein reales Sicherheitsrisiko darstellen
- wie legitime IAM‑Funktionen missbraucht werden können
- und weshalb „wir haben das schon eingeschränkt“ kein Sicherheitskonzept ist

Im Mittelpunkt stand nicht das Erstellen neuer Rechte,
sondern das **Zurückholen alter, bereits existierender Berechtigungen**.

---

## 🧠 Methodischer Ansatz

Der Einstieg war klar:
Ich hatte gültige AWS‑Credentials
und habe mit Pacu gearbeitet.

Statt nach klassischen Angriffspfaden zu suchen,
habe ich mir eine einfache Frage gestellt:

> Was darf diese Identität *offiziell* tun?

Mein Fokus lag dabei auf:
- IAM‑Berechtigungen
- Policy‑Historien
- und generelle enumeration
  
Lesen, Prüfen und Verstehen.

---

## 🔍 Reconnaissance

### Beobachtungen

Der IAM‑User wirkte zunächst harmlos:
- keine administrativen Rechte
- keine Möglichkeit, neue Policies zu erstellen
- keine offensichtliche Eskalationsfläche

Beim genaueren Blick fiel jedoch auf:
- bestehende Policies hatten **mehrere Versionen**
- ältere Versionen waren weiterhin vorhanden
- und es war erlaubt, diese Versionen wieder zu aktivieren

Formal gesehen:
keine neuen Rechte.
Praktisch gesehen:
eine Zeitmaschine.

### Schlussfolgerungen

Die entscheidende Erkenntnis war:
**Einschränken bedeutet nichts, wenn alte Zustände abrufbar bleiben.**

Die Umgebung war nicht falsch konfiguriert,
sie war **unvollständig aufgeräumt**.

---

## 🚪 Initial Access

Der Zugriff war von Beginn an vorhanden:
ein Low‑Privileged IAM‑User.

Der Einstiegspunkt war keine Schwachstelle,
sondern eine erlaubte Funktion:
das Zurücksetzen einer Policy auf eine frühere Version.

Es war keine Zauberei, sondern ich habe lediglich ein Zustand wiederhergestellt,
der eigentlich längst hätte entfernt sein müssen.

---

## 🔼 Privilege Escalation

### Analyse

Die Eskalation entstand nicht durch neue Rechte,
sondern durch alte.

Durch das Aktivieren einer früheren Policy‑Version
standen plötzlich Berechtigungen zur Verfügung,
die offiziell längst nicht mehr vorgesehen waren.

IAM selbst hat korrekt funktioniert.
Das Problem lag im Umgang mit der Historie.

### Entscheidungsfindung

Ich hätte ab da noch weiter machen können da mir nun **24 Eskalations-Methoden** offen standen.
Aber da das Ziel des Labs die **Privilege Escalation** selbst war, war dies nicht mehr nötig. 

Alte Policy‑Versionen sind kein Archiv.
Sie sind ein Risiko.
> **Das Zurücklassen alter Informationen kann sehr gefährlich werden.**
---

## 🏁 Ergebnis

- Wiederherstellung höherer IAM‑Berechtigungen
- Zugriffserweiterung ohne Policy‑Neuerstellung
- vollständige Eskalation über legitime IAM‑Mechanismen

Die Kette war simpel:

Low‑Priv User  
→ Alte Policy‑Version  
→ Rollback  
→ Erweiterte Rechte  

---

## 📚 Learnings & Reflexion

### Technisch
- Policy Versioning ist keine Historie, sondern eine Angriffsfläche
- Alte IAM‑Policy‑Versionen müssen aktiv gelöscht werden
- Einschränken ohne Aufräumen schafft Scheinsicherheit

### Methodisch
- Lesen und **verstehen** von Konfigurationen ist mächtig
- IAM muss als Lebenszyklus betrachtet werden, nicht als Zustand
- Sicherheitsannahmen sollten regelmäßig hinterfragt werden

### Persönlich
- Vergessene Policy‑Versionen könnten durchaus gefährlich sein
- IAM‑Hygiene‑Management darf nie vernachlässigt werden
- Weniger Technik, mehr Verständnis bringt oft die besseren Ergebnisse

---

👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-aws-iam-privesc-by-rollback-walkthrough-5b90deffaa80)**  

---

> 💡 **Hinweis:**  
> Dieses Write‑up beschreibt meinen Denk‑ und Lernprozess  
> und verzichtet bewusst auf eine technische Schritt‑für‑Schritt‑Anleitung.
