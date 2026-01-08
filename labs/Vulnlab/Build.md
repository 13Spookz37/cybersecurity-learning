# VulnLab – Build

> **Plattform:** VulnLab  
> **Kategorie:** CI/CD · interne Dienste · Angriffspfade  
> **Schwerpunkt:** Jenkins · Gitea · Pipeline-Designfehler  
> **Schwierigkeitsgrad:** mittel  
> **Status:** abgeschlossen  

---

## 1. Ausgangslage
**Kontext:** Build- und Entwicklungsumgebung mit mehreren erreichbaren Services.

Zu Beginn war unklar:
- welcher Service den Einstieg ermöglicht
- ob der Fokus auf Web, Infrastruktur oder internen Komponenten liegt
- wie stark CI/CD-Systeme eingebunden sind

---

## 2. Erste Annahmen
Ich bin zunächst von einem klassischen Service- oder Web-Angriff ausgegangen.  
CI/CD-Komponenten habe ich anfangs eher als unterstützend betrachtet.

> **Rückblick:**  
> Diese Annahme hat meinen Fortschritt verlangsamt.

---

## 3. Reconnaissance – Beobachtungen
In der Recon-Phase wurde deutlich:
- mehrere Dienste existieren parallel
- Backups sind zugänglich
- Build-Artefakte enthalten sensible Informationen

Der entscheidende Punkt war nicht *ein einzelner Dienst*, sondern **die Verbindung zwischen ihnen**.

---

## 4. Entscheidungsstellen
Eine zentrale Frage war:

> **Ist die CI/CD-Kette nur Kontext – oder der eigentliche Angriffspfad?**

Ich habe mich bewusst entschieden, sie als **primäres Ziel** zu betrachten.  
Automatisierung bedeutet Macht – besonders, wenn Vertrauen falsch gesetzt ist.

---

## 5. Fehler & Sackgassen
- Klassische Exploit-Denkmuster zu stark priorisiert
- Interne Dienste unterschätzt
- CI/CD-Security zu spät ernst genommen

Diese Fehler waren zeitintensiv, aber lehrreich.

---

## 6. Zentrale Learnings
- CI/CD-Pipelines sind hochprivilegierte Angriffsflächen
- Backups sind oft Einstiegspunkte
- Designfehler schlagen Exploits
- Interne Dienste sind meist entscheidend

---

## 7. Übertragbarkeit
Sehr realistisch für:
- Unternehmensnetzwerke
- DevOps-Umgebungen ohne Security-Fokus
- Red-Team- & Audit-Szenarien

---

## 8. Weiterführende Gedanken
- Wie hätten Logging & Monitoring geholfen?
- Welche Architektur hätte diesen Angriff verhindert?
- Wie früh hätte man diesen Pfad erkennen können?

👉 **Technischer Walkthrough:**  
[Medium](https://medium.com/@13spookz37/build-vm-walkthrough-b28c89d45c63)

---

> Dieses Writeup zeigt meinen Denk- und Entscheidungsprozess –  
> nicht nur die technische Umsetzung.
