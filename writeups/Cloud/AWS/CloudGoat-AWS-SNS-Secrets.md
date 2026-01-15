# Lab: CloudGoat AWS SNS Secrets

**Plattform:** CloudGoat  
**Schwierigkeit:** Easy  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · IAM · SNS · API Gateway  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Dieses Lab sollte zeigen, dass man in AWS auch ohne „starke“ Rechte
trotzdem an sensible Informationen kommen kann.

Es ging nicht darum:
- irgendwo Admin zu werden
- IAM auszutricksen
- eine Eskalation zu bauen

Sondern zu verstehen:
- wie Informationen in AWS verteilt werden
- warum Messaging-Services schnell unterschätzt werden
- dass ein API-Key alleine schon Zugriff bedeuten kann

Mein persönliches Ziel war es, besser zu verstehen,
**wie sich aus scheinbar harmlosen Leserechten echte Zugriffsmöglichkeiten ergeben**.

---

## 🧠 Methodischer Ansatz

Am Anfang bin ich davon ausgegangen, dass es wieder auf IAM hinausläuft.

Meine Erwartung:
- irgendwo eine Rolle
- irgendwo Trust
- irgendwo ein klassischer Eskalationspfad

Diese Denkweise hat mir hier nicht geholfen.

Stattdessen habe ich angefangen,
mir einfach anzuschauen:
- welche Services sichtbar sind
- wo Daten liegen
- was ich konkret lesen darf

Pacu und AWS CLI halfen mir dabei Schritt für Schritt zu verstehen, was da eigentlich passiert.

---

## 🔍 Reconnaissance

### Beobachtungen

- Der IAM-User hatte nur sehr eingeschränkte Rechte
- Keine Möglichkeit, Policies zu ändern
- Kein direkter Zugriff auf sensible Services

Was mir aber auffiel:
- SNS war zugänglich
- Topics konnten eingesehen werden
- Nachrichteninhalte waren lesbar

Das wirkte zuerst unspektakulär.

### Schlussfolgerungen

Ich habe mir dann die Frage gestellt:
- **Warum werden hier überhaupt Daten verteilt?**

Es ging nicht um klassische Exploits,
sondern darum, SNS zu verstehen.

---

## 🚪 Initial Access

Der Zugriff war bereits da:
Low-Privileged AWS Credentials.

Der eigentliche Einstieg war,
dass ich über SNS **Informationen bekommen habe,
die nicht für mich gedacht waren**.

Konkret:
- ein API-Key
- offen über einen Messaging-Service verteilt

Das war im grunde kein Hack.
Das war einfach die Logik dahinter.

---

## 🔼 Privilege Escalation

### Analyse

Es gab hier keine klassische Privilege Escalation.

Stattdessen:
- ein API-Key
- ein API-Gateway
- legitime Requests

IAM hat sich dabei nicht geändert.
Aber mein Zugriff schon.

### Entscheidungsfindung

Ich habe mich bewusst dagegen entschieden,
nach einer „besseren“ Eskalation zu suchen.
Zumal das Lab auch darauf ausgelegt war, diesen Weg zu gehen.

Der Punkt war:
**Der Zugriff war bereits ausreichend.**

Der Fehler lag nicht in fehlenden Schutzmechanismen,
sondern darin, **wie Secrets verteilt wurden**.

---

## 🏁 Ergebnis

- Zugriff auf einen geschützten API-Endpoint
- Nutzung eines API-Keys, der offen verteilt wurde
- Nachweis, dass Information Disclosure hier ausreicht

Die Kette war simpel:

Low-Priv User  
→ SNS-Nachrichten  
→ API-Key  
→ API-Gateway Zugriff  

---

## 📚 Learnings & Reflexion

### Technisch
- SNS ist kein „harmloser Notification-Service“, sondern kann bewusst oder unbewusst zum Transport sensibler Informationen (Secrets, Tokens, interne URLs) genutzt werden
- Ein gefundener API-Key oder Token ist bereits ein gültiger Zugriff und kein bloßes Konfigurationsdetail
- Auch bei sauber wirkenden IAM-Rollen können Daten exponiert werden, wenn sensible Informationen über SNS-Inhalte nach außen gegeben werden

### Methodisch
- Nicht jedes Lab führt zu Privilege Escalation – reines Lesen von Daten kann bereits das eigentliche Ziel sein
- Statt sofort nach komplexen Angriffspfaden zu suchen, lohnt es sich, zuerst die vorhandenen Services inhaltlich zu prüfen
- AWS-Services müssen im Zusammenspiel betrachtet werden: SNS, API Gateway und Umgebungsvariablen ergeben gemeinsam den Angriffspfad

### Persönlich
- Mein Fokus lag zu sehr auf Pacu - ich werde mir für die Zukunft eine hybride Arbeitsweise angewöhnen
- Mehr auf Details achten und die Architektur von AWS besser verstehen lernen
- Kleine Erfolge sind immer noch Erfolge


👉 **Technischer Walkthrough: [Medium](https://medium.com/@13spookz37/cloudgoat-aws-sns-secrets-walkthrough-65ffffe5cab5)**  

---

> 💡 **Hinweis:**  
> Dieses Write-Up beschreibt meinen Denkprozess  
> und verzichtet bewusst auf eine Schritt-für-Schritt-Anleitung.
