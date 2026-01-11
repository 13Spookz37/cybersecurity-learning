# Lab: AWS Privilege Escalation

**Plattform:** CloudGoat  
**Schwierigkeit:** Easy  
**Kategorie:** Cloud  
**Technologiefokus:** AWS · IAM · Privilege Escalation  
**Status:** Completed

---

## 🎯 Ziel des Labs

In diesem Lab steht eine CloudGoat AWS Umgebung im Fokus, in der ein eingeschränkter IAM-User vorhanden ist.

Das Ziel war nicht:
- einen offensichtlichen Bug zu finden
- einen bekannten Exploit auszuführen

Sondern zu verstehen:
- wie Rechte in AWS zusammenwirken
- welche Auswirkungen Trust-Beziehungen in IAM haben
- wie „harmloses“ Lesen und Rollenannahme zu kompletter Eskalation führen kann

---

## 🧠 Methodischer Ansatz

Die Ausgangssituation war klar definiert:
- ein IAM-User mit limitierten Rechten
- keine Admin-Rechte
- scheinbar keine direkte Eskalationsmöglichkeit

Meine erste Erwartung war eine klassische Cloud Fehlkonfiguration, die direkt ausgehebelt werden kann.

Diese Annahme war zu früh.

Statt sofort in Tools oder Exploit-Techniken einzusteigen, habe ich:
- die IAM-Rechte im Kontext gelesen
- nicht nur Permissions, sondern auch Trust-Policies analysiert
- verstanden, dass Rechte *in Kombination* gefährlich werden

---

## 🔍 Reconnaissance

### Beobachtungen

- Der IAM-User hatte nur wenige scheinbar harmlose Rechte
- Viele List und Get-Berechtigungen
- Eine einzelne Policy, die nicht direkt Admin-Rechte enthielt
- Keine expliziten Eskalationsberechtigungen in der User-Policy

Das sah zunächst nach einer Sackgasse aus.

Entscheidend war aber zu erkennen:
- welche Aktionen der User *erlaubt* war auszuführen
- welche Rollen-Policies *anderen* Ressourcen vertrauen
- dass Trust-Policies eine massive Rolle spielen

---

## 🚪 Initial Access

Der initiale Zugriff war bereits gegeben: der IAM-User war eingeloggt mit gültigen
Credentials.

Es ging nicht darum, diesen Zugriff zu verbessern.

Es ging darum zu prüfen:
- was diese Identität *effektiv* tun darf
- welche impliziten Möglichkeiten sich ergeben

Zugriff → nicht exploitable  
aber → **ein Einstiegspunkt in Denkpfade, die zu Eskalation führen**

---

## 🔼 Privilege Escalation

Nach der Analyse der IAM-Policy war klar:
- der User selbst hatte keine Admin-Rechte
- aber er durfte Rollen mittels `sts:AssumeRole` annehmen

Das ist der kritische Punkt:
Rechte + Trust = Eskalation.

Die Rolle, die übernommen werden konnte, hatte selbst umfangreiche Rechte
(inklusive Lambda-Erstellung und PassRole).

Der Weg zur Eskalation war dabei kein „Exploit“ oder „Bug“, sondern:
- Verständnis für Trust-Policies
- Nutzung legitimer, vorhandener Berechtigungen
- keine Manipulation der Infrastruktur

---

## 🏁 Ergebnis

- Kontrolle über eine Rolle mit erhöhten Rechten  
- Erstellung und Ausführung einer AWS Lambda-Funktion  
- Ausführung von Code mit admin-nahen Rechten  

Am Ende war der Weg nicht ein einziger „Exploit“, sondern eine
Kombination von:
- Rollenannahme
- Trust-Beziehungen
- vorhandenen legitimen Berechtigungen

---

## 📚 Learnings & Reflexion

### Technisch

- `sts:AssumeRole` selbst ist kein „Read-Befehl“ – es erlaubt die Übernahme einer Rolle; die Eskalation entsteht durch Vertrauen in die falsche Identität
- IAM-Trust-Policies sind kritischer als reine Permission Listen
- `iam:PassRole` + Lambda ist ein häufiger Eskalationsvektor

### Methodisch

- Rechte im Kontext überdenken
- Nicht nur Permissions lesen, sondern auch Trust-Modelle
- Wiederholung bekannter Techniken hilft wenig ohne Kontext

### Rückblickend

- Zu früh nach „Bugs“ gesucht, statt die Architektur zu lesen
- Zu spät erkannt, dass Trust-Policies der eigentliche Schlüssel sind
- Nächstes Mal: früher Modelle und Beziehungen hinterfragen

---

👉 **Technischer Walkthrough**:  [Medium](https://medium.com/@13spookz37/aws-privilege-escalation-walkthrough-f991a431c5bf?postPublishedType=repub)
---

> **Hinweis:**  
> Dieses Write-Up dokumentiert Denk- und Entscheidungsprozesse.  
> Es enthält bewusst **keine technischen Details oder reproduzierbaren Schritte.**
