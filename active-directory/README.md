## Active Directory Pentesting – Methodikübersicht
<img width="3462" height="742" alt="Active Directory Pentesting Attack Flow" src="https://github.com/user-attachments/assets/7a669d7e-8ae2-44d5-a89b-950fe47d8fa3" />



Dieses Diagramm zeigt die logische Struktur und die Zusammenhänge der einzelnen
Phasen eines Active‑Directory‑Pentests.

### Legende
- **Pfeile** zeigen typische Übergänge zwischen Phasen
- **Linien** zeigen inhaltliche Abhängigkeiten und Rückkopplungen
- **Gleichfarbige Blöcke** gehören zur selben methodischen Ebene


---

## Methodischer Ansatz

Dieses Repository bildet Active‑Directory‑Pentesting als **zusammenhängendes System**
ab.

Statt linearer Schritt‑für‑Schritt‑Anleitungen werden hier **Phasen und deren
Wechselwirkungen** betrachtet. Ziel ist es, ein **mentales Modell** dafür zu
entwickeln, wie sich **gesammelte Informationen, Zugriffsrechte und strukturelle
Eigenschaften einer Active‑Directory‑Umgebung** über mehrere Phasen hinweg
auswirken und sich gegenseitig beeinflussen.

Die Inhalte dienen als **methodische Referenz und Denkstruktur** – nicht als
ausführbare Anleitung.

---

## Phasenübersicht

Die Struktur orientiert sich an folgenden Phasen:

1. Reconnaissance  
2. Initial Access  
3. Execution  
4. Privilege Escalation  
5. Lateral Movement  
6. Persistence  
7. Domain Dominance  
8. Cleanup  

Die Phasen sind **nicht strikt linear** zu verstehen. Erkenntnisse aus späteren
Phasen können frühere Phasen erneut beeinflussen oder erweitern.
