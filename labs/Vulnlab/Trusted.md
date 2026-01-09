# Lab: Trusted

**Plattform:** VulnLab  
**Schwierigkeit:** Easy  
**Kategorie:** Chain  
**Technologiefokus:** Active Directory  
**Status:** Completed  

---

## 🎯 Ziel des Labs

Das Lab zeigt eine Active-Directory-Forest-Umgebung mit Parent und Child-Domain, in der die offensichtlichen Angriffsflächen bewusst gehärtet sind.

Ziel war nicht, eine einzelne Schwachstelle auszunutzen, sondern zu verstehen:
- wo Angriffsflächen entstehen, wenn AD selbst gut abgesichert ist
- wie Trust-Beziehungen praktisch missbraucht werden können
- warum alternative Einstiegspunkte oft entscheidender sind als direkte AD-Angriffe

---

## 🧠 Methodischer Ansatz

Der Start war klassisch:
- Zwei Domain Controller → Forest-Struktur
- Erwartung, dass zumindest ein Standard-AD-Angriff greift

Das war nicht der Fall.

Mehrere gängige Ansätze liefen ins Leere. Rückblickend hätte ich früher akzeptieren müssen, dass die Umgebung absichtlich so gebaut ist.

Statt weiter Zeit in Variationen desselben Ansatzes zu investieren, war ein Perspektivwechsel notwendig:
- Welche Komponenten gehören nicht zwingend auf einen Domain Controller?
- Wo weicht die Umgebung vom erwartbaren Design ab?

---

## 🔍 Reconnaissance

### Beobachtungen

- Die AD-Kernkomponenten wirkten konsistent gehärtet
- Keine offensichtlichen Fehlkonfigurationen
- Keine triviale Enumeration möglich
- Zusätzliche Dienste auf einem Domain Controller

Letzter Punkt war auffällig.  
Nicht wegen einer konkreten Schwachstelle, sondern wegen der Rolle, die das System eigentlich haben sollte.

### Schlussfolgerungen

- AD selbst ist hier nicht der Einstieg
- Die Angriffsfläche entsteht durch Integration zusätzlicher Komponenten
- Der Fokus sollte auf diesen Abweichungen liegen, nicht auf weiterem AD-Bruteforcing

---

## 🚪 Initial Access

Der initiale Zugriff erfolgte über eine sekundäre Komponente, nicht über Active Directory.

Das war weniger eine technische Überraschung als eine architektonische:
Ein Domain Controller übernimmt Aufgaben, die nicht zu seiner Kernfunktion gehören.

Diese Entscheidung erzeugt implizites Vertrauen –  
und genau dieses Vertrauen wurde ausgenutzt.

---

## 🔼 Privilege Escalation

Nach dem initialen Zugriff war die Situation klar:
- Die Child-Domain war kontrollierbar
- Die Frage war nicht ob, sondern wie sich diese Kontrolle ausweiten lässt

An diesem Punkt ging es nicht mehr um Exploits, sondern um:
- Trust-Beziehungen
- implizite Berechtigungen
- legitime Mechanismen mit sicherheitskritischen Auswirkungen

Die Eskalation in die übergeordnete Domäne ergab sich logisch aus der Struktur.

---

## 🏁 Ergebnis

- Kontrolle über die untergeordnete Domäne
- Missbrauch bestehender Trust-Beziehungen
- Ausweitung der Kontrolle auf Forest Ebene

Ab hier existieren innerhalb des Forests keine sinnvollen Trennlinien mehr.

---

## 📚 Learnings & Reflexion

### Technisch

- Härtung einzelner AD-Komponenten reicht nicht aus
- Zusätzliche Services auf Domain Controllern sind kritisch
- Trust-Beziehungen sind funktional notwendig, aber sicherheitlich gefährlich

### Methodisch

- Wiederholung desselben Ansatzes bringt keinen Fortschritt
- Auffällige Designentscheidungen sind oft relevanter als bekannte Angriffstechniken
- Strukturverständnis ist bei AD-Umgebungen entscheidend

### Rückblickend

- Zu viel Zeit in Standard-Enumeration investiert
- Zu spät akzeptiert, dass AD selbst nicht der Einstieg ist
- Nächstes Mal: früher Architektur hinterfragen

---

## 🔗 Weiterführende Ressourcen

👉 **Technischer Walkthrough** :

[Medium](https://medium.com/@13spookz37/trusted-vm-walkthrough-6ced3f350035)

---

> **Hinweis:**  
> Dieses Write-Up beschreibt den Entscheidungs- und Denkprozess.  
> Es enthält bewusst keine technischen Details oder reproduzierbaren Schritte.
