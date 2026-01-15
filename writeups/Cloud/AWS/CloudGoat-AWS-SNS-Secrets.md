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

Ich bin strukturiert in das Lab gestartet.

Pacu lief und mein Fokus lag direkt auf SNS.
Nach einer kurzen Orientierung habe ich gezielt nach SNS‑bezogenen Modulen gesucht
und mit der Enumeration begonnen.

Der technische Weg war mir also klar:
- Service identifizieren
- Inhalte auflisten
- verstehen, was zugänglich ist

Die eigentliche Herausforderung lag nicht darin,
*wie* ich vorgehe,
sondern *wie ich die Ergebnisse einordne*.

Erst beim Lesen der Inhalte wurde deutlich,
dass der Zugriff selbst bereits das Ziel ist
und keine weitere Eskalation notwendig war.


---

## 🔍 Reconnaissance

### Beobachtungen

Der IAM‑User war stark eingeschränkt:
- kaum relevante Rechte
- keine Möglichkeit, Policies zu verändern
- keine offensichtliche Eskalationsfläche

Parallel dazu war aber klar:
- SNS war zugänglich
- Topics ließen sich enumerieren
- Nachrichteninhalte konnten gelesen werden

Technisch wirkte das zunächst unspektakulär.
Kein Exploit, kein Bruch, kein offensichtlicher Fehler.

### Schlussfolgerungen

Der entscheidende Punkt war nicht,
dass bereits sensible Daten einsehbar waren.

Sondern:
- dass ein Subscribe möglich war
- dass SNS als Verteiler für Secrets genutzt wurde
- und dass zukünftige Nachrichten für diese Identität bestimmt waren

Der API‑Key lag nicht offen herum.
Er wurde erst durch die Subscription zugestellt.

Der sicherheitsrelevante Fehler lag also nicht im Inhalt,
sondern im Design:
wer Nachrichten empfangen darf – und warum.


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
