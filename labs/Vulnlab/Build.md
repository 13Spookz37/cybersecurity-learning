📄 VulnLab – Build

Plattform: VulnLab
Schwerpunkt: CI/CD-Misskonfigurationen, interne Services, Angriffspfade
Schwierigkeitsgrad: Easy (methodisch anspruchsvoll)
Status: abgeschlossen

1. Ausgangslage

Ziel des Labs war es, ausgehend von einem extern erreichbaren System schrittweise Zugriff zu erlangen und interne Dienste sinnvoll zu nutzen, um vollständige Kontrolle zu erreichen.

Zu Beginn war unklar:

welche Services tatsächlich relevant sind

ob der Fokus eher auf Web, Infrastruktur oder internen Komponenten liegen würde

wie stark CI/CD-Komponenten eingebunden sind

2. Erste Annahmen & Erwartungen

Meine anfängliche Erwartung war ein klassischer Web-oder Service-Exploit.
CI/CD-Systeme wie Jenkins oder Gitea habe ich zunächst eher als unterstützende Komponenten eingeordnet – nicht als primären Angriffsvektor.

Rückblickend war das eine falsche Gewichtung.

3. Reconnaissance – was ich gesehen habe

In der Recon-Phase wurden mehrere Dienste sichtbar, die auf den ersten Blick unabhängig voneinander wirkten.
Erst durch genaues Hinsehen wurde klar:

Es existiert eine Build-/Dev-Infrastruktur

Backups sind erreichbar

Credentials und Konfigurationsartefakte sind nicht ausreichend geschützt

Der entscheidende Punkt war nicht welcher Dienst lief, sondern wie sie logisch zusammenhängen.

4. Entscheidungsstellen

Eine zentrale Entscheidung war:

Behandle ich CI/CD nur als Nebenfund oder als Kern des Angriffspfads?

Ich habe mich bewusst dafür entschieden, die Build-Kette als Angriffsoberfläche zu betrachten:

Konfigurationen sagen oft mehr als laufende Services

Automatisierung ist mächtig – für Defender und Angreifer

Diese Entscheidung hat den gesamten weiteren Angriffspfad bestimmt.

5. Fehler & Sackgassen

Zu Beginn habe ich klassische Exploit-Denkmuster priorisiert

Interne Dienste habe ich zunächst unterschätzt

Erst spät wurde klar, wie kritisch Pipeline-Vertrauen ist

Diese Verzögerung hat Zeit gekostet, war aber lehrreich.

6. Zentrale Learnings

CI/CD-Pipelines sind hochprivilegierte Angriffsvektoren

Backups sind kein Nebenschauplatz, sondern oft der Einstieg

Credentials in Build-Artefakten sind realistische Fehlkonfigurationen

Root-Zugriff entsteht hier nicht durch Exploits, sondern durch Designfehler

Interne Dienste sind oft der eigentliche Schatz

7. Übertragbarkeit

Dieses Lab ist sehr realitätsnah für:

interne Unternehmensnetzwerke

schlecht segmentierte Build-Umgebungen

DevOps-Teams ohne Security-Awareness

Besonders relevant für:

Red-Team-Szenarien

Cloud- und Hybrid-Umgebungen

CI/CD-Security-Audits

8. Weiterführende Gedanken

Wie lassen sich solche Angriffspfade frühzeitig erkennen?

Welche Logging- und Monitoring-Mechanismen hätten hier geholfen?

Wie sollte eine sichere Pipeline-Architektur aussehen?

👉 Ausführlicher technischer Walkthrough:
[Medium](https://medium.com/@13spookz37/build-vm-walkthrough-b28c89d45c63)

Dieses Writeup dokumentiert meinen Denk- und Entscheidungsprozess.
Es zeigt warum etwas funktioniert hat – nicht nur dass es funktioniert.
