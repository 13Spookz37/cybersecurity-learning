# Lab: CloudGoat AWS Vulnerable Cognito

**Plattform:** CloudGoat  
**Schwierigkeit:** Medium  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · Cognito · JWT · Identity Management  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Dieses Lab hatte keinen klassischen Einstieg über Access Keys oder IAM-Rollen.  
Der Ausgangspunkt war lediglich eine **URL mit einer Login-Maske**.

Ziel war es zu verstehen,
- wie Cognito Identitäten und Tokens ausstellt
- warum fehlende oder falsche Validierung ausreicht, um Zugriff zu erhalten
- und weshalb „nur eine Login-Page“ bereits eine Angriffsfläche ist

Im Mittelpunkt stand nicht das Umgehen von Schutzmechanismen,
sondern das **Verstehen der Identitätslogik hinter der Anwendung**.

---

## 🧠 Methodischer Ansatz

Der Startpunkt war ungewöhnlich simpel:
keine Keys, keine CLI-Zugänge, keine bekannten Einstiegspunkte.

Ich habe mich deshalb nicht gefragt,
*wie* ich eskaliere,
sondern:
> **Was ist das hier überhaupt?**

Mein Vorgehen war bewusst pragmatisch:
- Login-Mechanismus anschauen
- Verhalten der Anwendung beobachten
- Tokens analysieren, die nach dem Login entstehen

Googeln, Lesen, ausprobieren, nachdenken.

---

## 🔍 Reconnaissance

### Beobachtungen

Die Anwendung wirkte auf den ersten Blick normal:
- einfache Login-Maske
- keine Hinweise auf AWS
- keine sichtbaren Fehlermeldungen

Nach dem Login fiel jedoch auf:
- es werden Tokens zurückgegeben
- diese Tokens sind JWTs
- und sie enthalten wichtige Informationen

Der eigentliche Fokus verlagerte sich damit weg von der Oberfläche
hin zu dem, **was im Hintergrund passiert**.

### Schlussfolgerungen

Cognito war hier nicht nur Authentifizierung,
sondern **der zentrale Kontrollpunkt**.

Die Frage war nicht:
„Kann ich mich anmelden?“  
sondern:
**„Was passiert mit meiner Identität nach dem Login?“**

---

## 🚪 Initial Access

Der Zugriff entstand logisch, durch akzeptierte Eingaben.

Die Anwendung hat mir nach dem Login Tokens ausgestellt,
die als Vertrauensanker dienten.

Ergo:
- legitime Anmeldung
- gültige Tokens
- und auch fehlende Einschränkung bei deren Nutzung

---

## 🔼 Privilege Escalation

### Analyse

Es gab hier keine klassische Privilege Escalation über IAM.

Stattdessen:
- Identität wird über Cognito definiert
- Berechtigungen hängen am Token
- und diese Tokens wurden zu großzügig akzeptiert

Der Zugriff wurde nicht erweitert,
er wurde **nicht ausreichend begrenzt**.

### Entscheidungsfindung

Hier gab es keine weitere Eskalation.

Das Lab zeigte klar:
**Der Fehler liegt nicht im Ausbau von Rechten,
sondern im Vertrauen in Identitätsinformationen.**

Wenn Tokens falsch validiert werden,
ist jede weitere Sicherheitsmaßnahme zweitrangig.

---

## 🏁 Ergebnis

- Zugriff auf geschützte Ressourcen
- Nutzung gültiger, aber zu mächtiger Tokens
- vollständiger Zugriff ohne klassische IAM-Eskalation

Die Kette war ungewöhnlich kurz:

Login-Page  
→ Cognito Token  
→ Vertrauen der Anwendung  
→ Zugriff  

---

## 📚 Learnings & Reflexion

### Technisch
- Cognito ist kein „Login-Service“, sondern ein Identitätsprovider mit direkter Sicherheitsrelevanz
- JWTs sind nur so sicher wie ihre Validierung
- Token-basierter Zugriff kann IAM vollständig umgehen

### Methodisch
- Nicht jeder Einstieg beginnt mit Access Keys
- Web-Oberflächen verdienen dieselbe Aufmerksamkeit wie Cloud-APIs
- Verstehen = Methodik

### Persönlich
- Ich habe gemerkt, dass ich Cognito selbst nur als Login-Baustein gesehen habe
- Der Fokus auf klassische IAM-Pfade hätte hier nicht geholfen
- Dieses Lab hat mein Verständnis von Cloud-Identitäten erweitert

---

👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-aws-vulnerable-cognito-walkthrough-35bb2cd6c8be)**  

---

> 💡 **Hinweis:**  
> Dieses Write-up beschreibt meine Beobachtungen und Gedankengänge  
> und verzichtet bewusst auf eine technische Schritt-für-Schritt-Anleitung.
