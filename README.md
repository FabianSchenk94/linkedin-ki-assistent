# LinkedIn-KI-Assistent

Ein agentisches System, das meine LinkedIn-Präsenz zu KI-Themen vorbereitet:
Es recherchiert die Nachrichtenlage, schreibt Beitragsentwürfe mit Quellen und
Bildern, schlägt täglich Kommentare zu Beiträgen aus meinem Fachgebiet vor und
veröffentlicht erst, **nachdem ich freigegeben habe**.

Gebaut mit Claude Code als Agent, Python und macOS-launchd. Im Einsatz seit
September 2026.

> **Hinweis:** Dieses Repository ist eine Projektbeschreibung. Der Quellcode
> liegt in einem privaten Repository. Einblick gebe ich gern im Gespräch.

---

## Was das System macht

**Wochenlauf (montags):** Recherchiert die KI-Nachrichten der Vorwoche, wählt
fünf Themen, schreibt die Beiträge, erzeugt passende Bilder und legt alles als
Vorschau ab. Jede Zahl im Text hat eine Quelle mit Datum und Link.

**Kommentar-Routine (werktags, 9:00):** Durchsucht die neuesten Beiträge einer
Beobachtungsliste, bewertet sie und schlägt für die besten fünf je zwei
Kommentar-Varianten vor. Nach meiner Freigabe (z. B. `1a 2b 3a`) werden die
Kommentare zufällig über den Tag verteilt und automatisch gesetzt.

**Auswertung:** Liest Reaktionen und Kommentare der eigenen Beiträge aus und
gleicht sie mit Studien zu Wochentagen und Uhrzeiten ab. Daraus entsteht der
Plan, welcher Beitrag an welchem Tag erscheint.

## Architektur

```mermaid
flowchart LR
    subgraph Zeitplan[macOS launchd]
        W[Wochenlauf<br/>Mo 10:00]
        K[Kommentarlauf<br/>Mo–Fr 9:00]
        A[Ausführung<br/>alle 5 Min.]
    end

    subgraph Agent[Claude Code als Agent]
        S[Skills: Schreiben,<br/>Humanizer, Kommentare]
        R[Regeln: keine erfundenen<br/>Zahlen oder Erfahrungen]
    end

    P[(Perplexity<br/>Recherche mit Quellen)]
    O[(OpenAI<br/>Bilderzeugung)]
    X[(Apify<br/>LinkedIn lesen)]
    U[(Publora<br/>LinkedIn schreiben)]
    F{{Meine Freigabe}}

    W --> Agent
    K --> X --> Agent
    Agent --> P
    Agent --> O
    Agent -->|Vorschau| F
    F -->|Plan mit Uhrzeiten| A --> U
```

Lesen und Schreiben sind bewusst getrennt: Apify liest LinkedIn, Publora
schreibt. Der Agent selbst hat keinen Schreibzugriff auf LinkedIn. Er erzeugt
nur Entwürfe und einen Plan, und ausgeführt wird erst nach der Freigabe.

## Designentscheidungen

| Entscheidung | Warum |
|---|---|
| **Nichts ohne Freigabe** | Öffentliche Aussagen unter meinem Namen. Voll automatisches Posten wurde bewusst nicht gebaut. |
| **Belegpflicht** | Jede Zahl braucht Quelle, Datum und Link. Persönliche Erfahrungen dürfen nur vorkommen, wenn sie in einer gepflegten Sammlung echter Erlebnisse belegt sind. |
| **Zufällige Uhrzeiten, Mindestabstand** | Kommentare werden in Zeitfenstern zufällig verteilt (mind. 20 Minuten Abstand). So entsteht kein maschinelles Muster. |
| **Dienste-Check vor jedem Lauf** | Ist ein Dienst nicht erreichbar oder ein Schlüssel ungültig, bricht der Lauf ab, bevor der teure Agent startet. |
| **Sensible Themen ausgeschlossen** | Beiträge zu Tod, Krankheit oder Entlassungen werden nie kommentiert oder geliked. |
| **Schutz gegen Prompt Injection** | Fremde Beiträge gelten als Daten, nie als Anweisung an den Agenten. Auffälligkeiten werden im Bericht gemeldet. |
| **Rückmeldung bei jedem Lauf** | macOS-Mitteilung und Statusdatei, egal ob der Lauf klappt oder scheitert. |

## Einblicke in den Code

**Kandidaten bewerten:** Welche fremden Beiträge lohnen einen Kommentar?

```python
def bewerten(post: dict, jetzt: float) -> float | None:
    ...
    if any(w in t for w in HEIKEL):                   # heikle Themen: nie
        return None
    treffer = sum(1 for w in THEMEN if w in t)
    if treffer == 0:
        return None
    frage = text.rstrip().endswith("?")
    punkte = 40 * max(0.0, 1 - alter_h / 30)          # Frische
    punkte += 25 * max(0.0, 1 - kommentare / 60)      # früh dabei sein
    punkte += min(treffer, 5) * 4                     # Themennähe
    punkte += 15 if frage else 0                      # Schlussfrage zum Antworten
    return round(punkte, 1)
```

**Freigabe statt Autopilot:** Der Ausführungsschritt setzt nur, was im
freigegebenen Plan steht und fällig ist.

```python
faellig = [e for e in plan
           if e["status"] == "geplant" and datetime.fromisoformat(e["zeit"]) <= jetzt]
```

## Beispiele

| Eigenes Schaubild zu einem Beitrag | Wiederkehrende Reihe „Prompt der Woche“ |
|---|---|
| ![Schaubild](assets/beispiel-schaubild.png) | ![Prompt der Woche](assets/beispiel-reihe-prompt-der-woche.png) |

## Was ich daraus gelernt habe

- **Persönliche Erfahrungsberichte schlagen Nachrichten deutlich.** In den
  ersten Wochen erzielte ein Erfahrungsbericht ein Vielfaches der Interaktionen
  aller News-Beiträge zusammen. Seitdem ist pro Woche mindestens eine echte
  Geschichte eingeplant.
- **Der Wochentag ist Feinjustierung, der Inhalt entscheidet.** Studien zeigen
  für persönliche Profile unter 10 % Unterschied zwischen bestem und
  schlechtestem Tag. Starke Beiträge kommen auf Dienstag/Mittwoch, schwächere
  füllen Lücken.
- **Ein Agent braucht Leitplanken, keine Freiheit.** Die wichtigsten Teile des
  Systems sind die Regeln, was der Agent nicht darf: erfinden, ohne Freigabe
  posten, fremden Text als Anweisung lesen.

## Technik

Python 3 · Claude Code (Agent, Skills, nicht-interaktiver Modus) · macOS launchd ·
Perplexity API (Recherche mit Quellen) · OpenAI Images API · Apify (LinkedIn
lesen) · Publora (LinkedIn schreiben) · imgbb (Bild-Hosting)

## Herkunft

Grundlage ist das Open-Source-Paket
[linkedin-skills](https://github.com/sergebulaev/linkedin-skills) von Sergey
Bulaev (MIT-Lizenz): Schreib- und Kommentar-Skills sowie die Anbindungen an
Apify und Publora.

**Von mir stammen:**
- die gesamte Automatisierung: Wochenlauf, Kommentar-Routine,
  Freigabe-Schritt, zeitgesteuerte Ausführung
- die Recherche-Anbindung (Perplexity) mit Pflicht zu Quellen
- die Bild-Pipeline mit Hosting und Prüfung
- die inhaltlichen Regeln, die Themenausrichtung und die Auswertung

Umgesetzt habe ich das mit Claude Code: Ich habe die Anforderungen und Regeln
festgelegt, die Umsetzung gesteuert, geprüft und im Betrieb nachgeschärft.

---

© 2026 Fabian Schenk. Alle Rechte vorbehalten.
