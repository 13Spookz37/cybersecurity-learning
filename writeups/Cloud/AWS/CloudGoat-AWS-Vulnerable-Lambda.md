# Lab: CloudGoat AWS Vulnerable Lambda.md

**Plattform:** CloudGoat  
**Schwierigkeit:** Medium  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · Lambda · IAM · SQL Injection  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Der Einstieg in dieses Lab war klassisch:
ein IAM-User mit minimalen Rechten.

Ich bin nicht davon ausgegangen,
direkt auf eine kritische Fehlkonfiguration in IAM zu stoßen.
Vielmehr ging es mir darum zu verstehen,
**wie weit man mit scheinbar harmlosen Rechten tatsächlich kommt**.

Die Lambda-Funktion stand dabei nicht am Anfang,
sondern wurde erst im Verlauf des Labs relevant.

---

## 🧠 Methodischer Ansatz

Ich bin mit einem Low-Privileged IAM-User gestartet
und habe zunächst klassisch enumeriert:
- welche User gibt es
- welche Aktionen erlaubt sind
- und wo sich Einstiegspunkte ergeben könnten

Erst im weiteren Verlauf fiel die Lambda-Funktion auf.

Nicht als direkter Angriffspunkt,
sondern als Bindeglied
zwischen IAM-Rechten,
Datenbankzugriff
und Anwendungslogik.

Ab diesem Moment lag der Fokus nicht mehr auf IAM allein,
sondern auf dem Zusammenspiel der Komponenten.

---

## 🔍 Reconnaissance

### Beobachtungen

Die Lambda-Funktion:
- verarbeitet Benutzereingaben
- greift auf eine Datenbank zu
- läuft mit einer IAM-Rolle, die mehr darf als sie sollte

Die Datenbankabfragen waren:
- dynamisch
- nicht sauber gefiltert
- direkt von externen Parametern abhängig

Kein komplexes Setup.  
Kein exotischer Bug.  
Einfach **klassische SQL Injection – in Serverless-Form**.

### Schlussfolgerungen

An diesem Punkt war klar:
Das Problem ist nicht Lambda.  
Das Problem ist **Vertrauen in die Funktion**.

Alles, was nach der Lambda passiert,
passiert **in ihrem Sicherheitskontext**.

Und der war mächtig.

---

## 🚪 Initial Access

Der Initial Access war von Beginn an gegeben:
ein IAM-User mit sehr eingeschränkten Rechten.

Keine Admin-Policies.
Keine direkten Eskalationsmöglichkeiten.
Kein offensichtlicher Fehltritt in IAM.

Der entscheidende Punkt war,
dass dieser User Zugriff auf eine Lambda-Funktion hatte,
deren interne Logik nicht ausreichend abgesichert war.

Der eigentliche Weg entstand also nicht beim Login,
sondern **nachgelagert – innerhalb der Funktion**.

---

## 🔼 Privilege Escalation

### Analyse

Die Privilege Escalation entstand **nicht durch IAM direkt**,
sondern durch eine SQL Injection innerhalb der Lambda-Funktion.

Diese Funktion lief mit einer IAM-Rolle,
die deutlich mehr Rechte hatte als der ursprüngliche User.

Durch die manipulierte Datenbankabfrage
konnte ich mich effektiv aus dem Kontext des Low-Priv Users lösen
und in den Berechtigungsrahmen der Lambda-Rolle wechseln.
> Eskalation durch Kontextwechsel, nicht durch Rollenänderung

Ab diesem Punkt war die Eskalation vollständig.

### Entscheidungsfindung

Ab dem Moment war die Eskalation abgeschlossen.

Nicht, weil nichts mehr möglich gewesen wäre,
sondern weil das Ziel erreicht war:
**vollständige Kontrolle durch eine einzige Schwachstelle**.

---

## 🏁 Ergebnis

- Administrative Berechtigungen über SQL Injection
- Eskalation ohne direkten IAM-Missbrauch
- Komplette Kontrolle durch eine einzige Lambda-Funktion

Die Kette war kurz:

Externer Input  
→ SQL Injection  
→ Lambda-Rolle  
→ Admin  

Kein komplexer Angriff.  
Nur falsches Vertrauen in eine „kleine“ Komponente.

---

## 📚 Learnings & Reflexion

### Technisch
- SQL Injection ist auch in Serverless-Architekturen tödlich
- Lambda-Rollen müssen so behandelt werden wie Root-Zugänge
- Externe Inputs in AWS-Services sind **kein Sonderfall**

### Methodisch
- Nicht Services angreifen, sondern **Datenflüsse verstehen**
- Kleine Komponenten haben oft den größten Impact
- „Das ist nur eine Lambda“ ist ein gefährlicher Gedanke

### Persönlich
- Ich war erstaunt, wie schnell man über Anwendungslogik zu Admin wird
- Dieses Lab hat mir klar gemacht, dass Serverless kein Sicherheitsgewinn ist
- Eine einzige schlecht geschriebene Query reicht aus

---

👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-aws-vulnerable-lambda-walkthrough-4daa39e5317c)**  

---

> 💡 **Hinweis:**  
> Dieses Write-up beschreibt meine Beobachtungen und Gedankengänge
> und verzichtet bewusst auf eine technische Schritt-für-Schritt-Anleitung.
