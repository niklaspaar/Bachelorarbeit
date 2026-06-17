# Optimierer-Infos: Fachliche Details aus Quelldokumenten

Zusammenfassung aller technischen und fachlichen Informationen aus den Quelldokumenten zu den Optimierern von SYNCROTESS BM.
Quellen: FIFO-PDFs (2023/2024), MIXOR-Dokumentationen, CoFlo-PPTX, RT-Anbindung MIXOR, Betonoptimierung-Anforderungen.

---

## 1. Überblick: Die Optimierer im Einsatz

SYNCROTESS BM verwendet zwei Hauptoptimierer für den Baustoff-/Transportbereich:

| Optimierer | Domäne | Algorithmus-Familie |
|------------|--------|---------------------|
| **FIFO / TRANSFER** | Transportbeton (Fertigbeton, Betonoptimierung) | Heuristik: Plant Dispatch + Transformationen |
| **MIXOR** | Transportbeton – Nachfolger von TRANSFER | Neue OR-basierte Heuristik, Standalone |
| **CoFlo** | Zementoptimierung (FTL, Massengut) | Heuristik + Set Partitioning MIP |

---

## 2. FIFO / TRANSFER – Transfer-Optimierung (Transportbeton)

**Quellen:** `20230531_TransferOptimization_engl.pdf`, `20241014_TransferOptimization_engl.pdf`

### 2.1 Zielsetzung

- **Primärziel:** Alle Lieferungen mit möglichst geringer Verspätung einplanen
- **Sekundärziele:**
  - Unnötige Leerfahrten vermeiden
  - First-in-First-Out (FiFo)-Regel einhalten: Erstes Fahrzeug am Werk bekommt die nächste Ladung
- **Allgemein:** Minimierung der offenen Lieferungen und der Verspätung; berücksichtigt Sonderfälle für den frühen Morgen (First Round)

### 2.2 Algorithmus-Ansatz (zweistufige Heuristik)

Exakte Methoden sind zu rechenintensiv → heuristischer Ansatz mit zwei Phasen:

**Phase 1: Plant Dispatch**
- Arbeitet auf Werksbasis: Fahrzeuge am Werk bedienen Aufträge am Werk (Hauptwerk wird bevorzugt, falls verfügbar)
- Einhaltung der FiFo-Regel
- Verspätungen möglich

**Phase 2: Transformationen** (optional, gesteuert via DG-Parameter `theEnableTrafos`)
- Tausch von Arbeit (Lieferungen) und/oder Ressourcen (Fahrzeuge) zwischen Werken
- Ziel: Offene und verspätete Lieferungen reduzieren, ohne übermäßige Fahrzeiten zu verursachen
- **4 Transformationstypen:**
  1. **Alternative Plant (AP):** Direkte Lieferung vom Anbieter-Werk
  2. **Approach from Plant (AfP):** Fahrzeugwechsel zu Empfänger-Werk, dann Lieferungen von dort
  3. **Approach from Customer (AfC):** Fahrzeugfahrt nach Lieferung mit Lieferungen vom Empfänger-Werk
  4. **Approach by Delivery (AbD):** Fahrzeugfahrt + Alternative Plant durch direkte Lieferung vom Anbieter und Fahrt zum Empfänger

**Ergebnis:** Zeitplan mit Informationen zu jeder Lieferung (Ladezeiten, Werk, Fahrzeug) → Proposals

### 2.3 Eingabedaten (Input)

- **Werke (Plants)**
- **Fahrzeuge/LKW**
- **Aufträge (Orders):**
  - Menge (min, max)
  - Kontinuierlicher Lieferfluss und Ladeabstand (Load-Spacing)
  - Kunde
  - (Zugewiesene) Lieferungen mit Lieferzeitintervallen
  - Hauptwerk (meist nächstes Werk) und Alternativwerke

### 2.4 Nebenbedingungen (Constraints)

- Schicht- und Arbeitszeiten (Single-Shift-Funktionalität, Planung für einen Tag einer Dispatch-Gruppe)
- FiFo-Regel an Betonwerken mit Ausnahmen
- First-Round-Regel: Abweichung von der geplanten Startzeit am Morgen wird berücksichtigt (Fairness)
- Prioritäten und Lieferzeitfenster von Aufträgen (definiert über Start der ersten Lieferung und Lieferfluss)
- Kontinuierlicher Lieferfluss mit festgelegter Menge pro Zeiteinheit und Vorgängeraufträgen mit Offset
- Verspätungen sind erlaubt – nichtlineare Kostenfunktion
- Frühes Laden (Early Loading) erlaubt
- Eigenschaften/Ausschlüsse von Fahrzeugen/Schichten, Aufträgen und Werken
- Fahrzeugkapazitäten und -prioritäten sowie Plusloads
- Öffnungszeiten von Werken, Park- und Kundenstandorten
- Maximale Kapazität pro Betonwerk
- Anzahl Ladeslots an Werken (Legs) und Alternativwerken
- Reinigungsdauern und feste/variable Entladedauern

### 2.5 FiFo-Regel im Detail

- Erstes Fahrzeug am Werk bekommt die nächste Lieferung
- FiFo-Regel ist sehr restriktiv → schnelle Zuweisung von Fahrzeugen und Lieferungen
- Berücksichtigt Fahrzeuge, die in der Vergangenheit an den Werken angekommen sind → Leerlaufzeiten
- Berücksichtigt auch Fahrzeuge, die in der Zukunft ankommen werden (erwartete Ladezeitpunkte)
- **Ausnahmen von der FiFo-Regel:**
  - Eigenschaften (Fahrzeug und Lieferung passen nicht zusammen)
  - First-Round-Regel
  - Hochprioritäre Aufträge

### 2.6 First Round und Morning Roster

- Ausnahmeregel für die erste Arbeitszuweisung des Tages
- Bevorzugt Fahrzeuge, die noch nicht gearbeitet haben (`thePreferNotYetWorked`)
- Disponent bestimmt Morning Roster (MR) am Werk, der die Reihenfolge der geplanten Fahrzeuge definiert
- **Realtime-Optimierung:**
  - Planung von Fahrzeugen mit SAS (Assigned Sign-On)
  - Keine Proposals bis Fahrzeug eingeloggt ist
  - Spätes Sign-On führt zur Bevorzugung des verspäteten Fahrzeugs
  - Frühes Sign-On führt nicht zur Bevorzugung (Ausnahme: Verspätung kann vermieden werden)

### 2.7 Fahrzeugprioritäten

- Fahrzeuge mit höherer Priorität 1 werden gegenüber Priorität 2 bevorzugt, wenn:
  - Mehrkosten durch höherprioritiertes Fahrzeug nicht überschritten werden
  - Morning Roster nicht verletzt wird
  - Höherprioritiertes Fahrzeug bis erwartetem Ladezeitpunkt des niederprioritären Fahrzeugs ankommt (Parameter `theTruckHoldingTimePrio1` vs. `theTruckHoldingTimePrio2`)

### 2.8 Early Loading (Frühes Laden)

- Werkkapazität = Anzahl Lieferungen pro Stunde oder Menge pro Zeiteinheit
- Bei Überlapp zweier Lieferungen: Entweder Verzögerung oder Frühes Laden (bevorzugt)
- Maximales frühes Laden via DG-Parameter `theDefaultShelfLife`
- Einschränkungen: Ankunft des geplanten Fahrzeugs, Startzeit der Optimierung, feste Lieferungen

### 2.9 Kostenfunktion

**Gesamtkosten einer Transformation t:**
> Kostendifferenz beim Anbieter + Kostendifferenz beim Empfänger + Kosten der Transformation t (zusätzliche Fahrt)

**Werkskosten bestehen aus:**
- Verspätung (Lieferung) – Hauptfaktor
- Frühes Laden (Lieferung)
- Überstunden (Fahrzeug)
- Leerlaufzeiten (Fahrzeug)

**DG-Parameter für Gewichtung:**
| Parameter | Bedeutung |
|-----------|-----------|
| `theScaleEarlyLoad` | Gewicht für frühes Laden |
| `theScaleIdle` | Gewicht für Leerlaufzeiten am Werk |
| `theScaleOvertime` | Gewicht für Überstunden am Werk |
| `theCapIdle` | Minimum für Leerlaufzeiten |
| `theCapOvertime` | Minimum für Überstunden |

**Verspätungskosten:** Quadratischer und linearer Teil mit $d$ = Verspätung:
$$\text{Kosten} = \min(m \cdot d + n, c \cdot d^2)$$

| Priority | Parameter c |
|----------|-------------|
| 0, 1, 5, 6 | `theScaleSquare1` |
| 2 | `theScaleSquare2` |
| 3 | `theScaleSquare3` |

---

## 3. MIXOR – Next-Generation Transportbeton-Optimierung

**Quellen:** `20260414_MIXORIntroductionIntern.pptx`, `20221130_FifoMixorComparison.pptx`, `20220922_MixorFunctionality_v0.1.docx`, `Betonoptimierung_v3.5.docx`

### 3.1 Einführung und Motivation

MIXOR = **Mix**ed **O**perations **R**esearch

**Warum MIXOR?**
MIXOR ist der neue Nachfolger des TRANSFER-Algorithmus (FIFO). Probleme mit TRANSFER:
- Erlaubte Verletzungen spezifischer Funktionalitäten
- Lange Rechenzeiten möglich
- Hoher Wartungsaufwand

MIXOR berücksichtigt aktuelle Forschungsergebnisse und erweitert den Funktionsumfang als intern finanziertes Projekt.

### 3.2 Vergleich TRANSFER vs. MIXOR

| Feature | TRANSFER | MIXOR |
|---------|----------|-------|
| Pausenplanung (statisch, dynamisch) | Nein | Ja |
| Mehrschicht-Planung | Nein | Ja |
| Lieferreduzierung durch Kapazitätsbetrachtung | Nein | Ja |
| Verbesserung der Fahrzeugauslastung | Nein | Ja |
| Verletzungen von Anforderungen | Hoch | Gering |
| Konfiguration der First-Round-Regeln | Einzeln | Mehrfach |
| Linksbündige Anfahrt mit WebGUI | Nein | Ja |
| Vorplanung ohne Roster | Nein | Ja |
| Realtime mit Roster | Nein | Ja |
| Optim auf Fahrzeugen in Realtime | Nein | Ja |
| Fahrzeugreduktionsmodus | Nein | Ja |
| Standalone-Implementierung | Nein | Ja |
| Wartungs- und Erweiterungsaufwand | Hoch | Gering–Mittel |
| Frühes Laden für erste Ladung je Schicht | Ja | Nein |

### 3.3 Leistungsvergleich (23 Szenarien, Heidelberg Materials)

| KPI | TRANSFER | MIXOR | Differenz |
|-----|----------|-------|-----------|
| Gesamte Fahrtzeit | 6.598:15:44 h | +105:37:10 h | MIXOR etwas mehr |
| Offene Menge (m³) | 392 m³ | -70 m³ | MIXOR besser |
| Gesamte Verspätung | 454:20:39 h | -40:10:09 h | MIXOR besser |
| CPU-Zeit | 38:43 min | 5:14 min | **MIXOR ~7× schneller** |
| Verarbeitete Menge (m³) | 61.894 | 61.903 | fast gleich |
| Pünktliche Lieferungen | 6.447 | 6.676 | MIXOR besser |
| Rote Lieferungen (krit. Verspätung) | 421 | 408 | MIXOR etwas besser |

**Fazit:** MIXOR ist deutlich schneller (ca. 7× geringere CPU-Zeit) bei gleichzeitig besserer Qualität in den meisten KPIs.

### 3.4 MIXOR-Algorithmus: Plant Dispatch

**PlantDispatcher-Pseudocode:**

1. Fahrzeuge am Werk: Sortierzeit auf Verfügbarkeit am Werk setzen
2. Fahrzeuge am Werk sortieren nach ihrer Sortierzeit
3. Für jedes Fahrzeug: `mNextDelivery` (nächste Lieferung) auf NULL setzen
4. **Iteration über Fahrzeuge** (wähle erstes Fahrzeug T1 aus der Liste):
   - Wenn `mNextDelivery` von T1 nicht NULL:
     - Wenn Lieferung bereits verplant (anderes Fahrzeug kam früher zurück): Suche erste zulässige Lieferung D1 (nach Ladezeit sortiert)
     - Sonst (noch nicht verplant): Lade- und Rückkehrzeit neu berechnen
       - Wenn Zeiten identisch: Lieferung auf Fahrzeug verplanen, Sortierzeit auf Rückkehrzeit setzen
       - Wenn Zeiten abweichen: Fahrzeug neu einsortieren oder andere Lieferung suchen
   - Wenn `mNextDelivery` von T1 NULL: Suche erste zulässige Lieferung D1
5. Abbruch wenn: kein nächstes Fahrzeug verfügbar, alle Iterationen durchlaufen, Lieferung muss verschoben werden, oder Zeitlimit erreicht

**PlantDispatch-Koordination (äußere Schleife):**

1. Sortiere PlantDispatches nach Ladezeit der nächsten Lieferung
2. Iteriere über PlantDispatches, rufe ersten auf (PD_n) mit Zeitlimit = Ladezeit der ersten Lieferung des nachfolgenden PD_{n+1}
3. Führe PD_n bis Zeitlimit oder bis Lieferungsverschiebung aus
4. Sortiere PD_n neu ein
5. Falls Lieferung zu anderem Werk PD_m verschoben: Auch PD_m neu einsortieren

### 3.5 MIXOR Architektur: Standalone & JSON-basiert

- MIXOR ist als **Standalone-Implementierung** realisiert (kein Monolith)
- Input/Output: **JSON-Dateien** (Black-Box-Ansatz)
- "**Optimization as a Service**" – entkoppelt von SYNCROTESS-Kernsystem
- Kann unabhängig deployed und gewartet werden
- Verschiedene Planungsmodi:
  - Vorplanung (Preplanning) für den nächsten Tag
  - Realtime-Optimierung

### 3.6 Mehrschicht-Planung

- MIXOR unterstützt Mehrschicht-Betrieb einschließlich Nachtschichten
- Alle Optim-Schichten im Fahrzeugkalender werden genutzt
- Bricht auch über Nacht hinaus durch

### 3.7 Pausenplanung (Break Planning)

**Statisch:** Eine „Mittagspause" festgelegter Länge im vordefinierten Zeitfenster

**Dynamisch:** Bis zu vier Pausen mit definierter Länge, frühestem Start und spätestem Ende

**Realtime-Anbindung** (aus `RT-Anbindung MIXOR v0.13.docx`):
- Drei Fälle der Pausenbehandlung:
  - **Fall 1:** Pausen werden vorgeschlagen und durch Disponenten angewiesen (auch automatisch möglich)
  - **Fall 2:** Pausen werden vorgeschlagen (Platzhalter im Plan), aber Fahrer selbst verantwortlich (keine automatischen Anweisungen)
  - **Fall 3:** Optimierung schlägt keine Pausen vor
- Pausendarstellung in der GUI: Gantt-Ansicht mit Farbkodierung (TS=grün, AS=blau, PS=grau)
- Pausendarstellung im Report & Assign-Fenster: Tassen-Symbol als Overlay in erster Spalte
- **Optimierungs-Ergebnisse zurückschreiben:** Optim-Schedule (OS) wird in Tour-Schedule (TS) überführt

### 3.8 Roadmap und Go-Live (Stand April 2026)

| Milestone | Verantwortlich | Datum |
|-----------|---------------|-------|
| Delivery Transformations verfügbar (AUS Sandpit) | INFORM | 06/2025 |
| Workshop (BE, NL, AUS) | HM + INFORM | 26.06.2025 |
| Truck Transformations verfügbar (SYNCROTESS 10.1.5) | INFORM | 10/2025 |
| Go-Live-Planung (Realtime) | HM + INFORM | offen |
| Workshop Geiger | – | 10/2025 |
| Workshop Unicon | – | 12/2025 |
| Go-Live Geiger | – | 05/2026 |
| Go-Live Unicon | – | 05/2026 |

**Kunden:** Heidelberg Materials (NL, BE, AUS), Geiger, Unicon, Polpaico, Gammon

---

## 4. Betonoptimierung: Anforderungen und Umsetzung (MIXOR als Nachfolger)

**Quelle:** `Betonoptimierung_v3.5.docx`

### 4.1 Abgelöste Algorithmen

MIXOR ersetzt zwei ältere Verfahren:

**Transfer-Algorithmus (FIFO):**
- Kann keine Mehrschichten verarbeiten (Probleme mit Multi-Schicht und Nachtschichten)
- Probleme mit festen Proposals (SASe konnten nicht berücksichtigt werden)

**Regret-Algorithmus:**
- Erzeugt für Disponenten unverständliche Ergebnisse
- Keine Pausenplanung

### 4.2 MIXOR-Kernprinzipien

- **FiFo-Regel:** Fahrzeuge vom selben Werk werden in FIFO-Reihenfolge verteilt (erstes verfügbares Fahrzeug = erstes verteilt)
- MIXOR erstellt je Werk:
  - Sortierte Liste der verfügbaren Fahrzeuge
  - Sortierte Liste der Lieferungen
- **Lieferungen** werden aufsteigend nach der linken Grenze des Ladefensters sortiert
- **Fahrzeuge** werden aufsteigend nach ihrer Verfügbarkeit am Werk sortiert

### 4.3 Technische Integration

- Entkoppelt von SYNCROTESS als „Optimization as a Service" (Black Box)
- JSON-basierte Kommunikation (Input: `ftlopt1.json`, Output: `ftlopt1.sol.json`)
- Aufruf: `ftlopt -i ftlopt1.json -w /tmp/workdir/ -s ftlopt1.sol.json`
- Unterstützt manuelle Proposals (SASe) und minutengenaue Planung

### 4.4 Parameter

- `Max Transfer Limit`: Steuert maximale Transferdistanz zwischen Werken
- `Plant Dispatch Only`: Verhindert Transferfahrten (nur werksinterner Einsatz)

### 4.5 Mehrschicht-Unterstützung

- Vollständige Unterstützung inkl. Übernacht- und Nachtschichten
- Fahrzeuge werden schichtübergreifend verplant

---

## 5. CoFlo – Zementoptimierung (Full Truck Load)

**Quelle:** `20220510_Coflo 1.pptx` (INFORM GmbH, Cement Optimization)

### 5.1 Domäne und Zielsetzung

CoFlo optimiert den **Zementtransport** (FTL, Full Truck Load / Massengut):
- Vollständig andere Problemdomäne als Transportbeton
- Ziel: Minimierung von offenen Lieferungen, Verspätungen, Leerlaufzeiten

**Optimierungsziele (konfigurierbar):**
- Offene Lieferungen minimieren
- Leerfahrten minimieren
- Verspätungen minimieren
- Vergeudete Arbeitszeit minimieren
- Kurze Lieferzeitfenster bevorzugen

### 5.2 Algorithmus-Ansatz

**Zwei-Phasen-Ansatz:**

**Phase 1: Heuristik (Fill Approach)**
- Eingabe: alle Fahrzeuge, Aufträge, Constraints
- Heuristisch: Auswahl einer Auftragsuntermenge
- Planung des Ergebnisses unter Berücksichtigung aller Constraints
- Berechnung der Kostentabelle unter Berücksichtigung aller Constraints
- **Core Algorithm:** Linear Assignment Problem (LAP)
  - Kostentabelle: Fahrzeuge × Aufträge (Distanzmatrix)
  - Kostentabelle muss nach jeder Iteration neu berechnet werden

**Phase 2: Extension – Set Partitioning (SP)**
- CoFlo generiert eine Anzahl von Lösungen = unabhängige Fahrzeugrouten (eine Route pro Fahrzeug)
- Ein Auftrag kann von maximal einem Fahrzeug bedient werden
- Speichert alle generierten Routen für jedes Fahrzeug
- Formulierung als **Mixed-Integer-Programming (MIP):**

$$\min \sum_{s \in S} \sum_{r \in R} c_{r,s} x_{r,s} + M \sum_{o \in O} y_o$$

subject to:
$$\sum_{s \in S} \sum_{r \in R} a_{s,r,o} x_{s,r} + y_o = 1, \quad \forall o \in O$$
$$\sum_{s \in S} x_{s,r} \leq 1, \quad \forall r \in R$$
$$x_{s,r} \in \{0,1\}, \quad y_o \in \{0,1\}$$

Löser: **SCIP** oder **CBC** (Open-Source MIP-Solver)

### 5.3 Nebenbedingungen (Constraints)

- Schichtzeiten der Fahrzeuge
- Öffnungszeiten der Standorte
- Auftrags-Zeitfenster
- Auftrags-Deadlines
- Ladedauern und Entladedauern
- Vorausladungen (Preload orders) und Vorausfahrten (Predrive orders)
- Arbeitsdauer der Schichten
- Fahrdauer der Schichten
- Roaming-Fahrzeuge und stationierte Fahrzeuge
- Geladene Parkplätze (Loaded parking)
- Prioritäten (Auftrags- und Fahrzeugprioritäten)
- Ausschlüsse und Eigenschaften
- Kapazitäten
- SAS-Handling (Assigned Sign-On)
- FAS-Handling

### 5.4 Wichtige Parameter

| Parameter | Beschreibung |
|-----------|-------------|
| `Max Empty Approach` | Maximale Leerfahrt zum Depot: max(Homerun, MaxEmptyApproach); Standard 220 km; nur für Nicht-Roaming-Fahrzeuge |
| `Delay Cost Threshold` | Schwellwert für Verspätungskosten (quadratisch bis Schwellwert, dann linear) |
| `Order Priority Weight` | Gewicht für Auftragspriorität; Standard 100 Sekunden |
| `Truck Priority Weight` | Gewicht für Fahrzeugpriorität; Standard 100 Sekunden |
| `Minimum Roaming Travel` | Roaming-Fahrzeug bleibt am Standort wenn weniger Zeit als Parameter übrig; Standard 3.600 s |
| `Truck Treated As Idle` | Fahrzeug wird als Leerlauf behandelt wenn letzte Proposal älter als Parameter; Standard 7.200 s |
| `Preloads Aligned Right` | Vorausladungen rechtsbündig ausrichten |
| `Extra Time Ratio For Preload` | Zusätzliche Zeit für Vorausladungen |
| `Prefer Early Loading` | Frühes Laden bevorzugen |
| Kostenfunktions-Gewichtungen | Verschiedene Gewichte für Kostenkomponenten |

---

## 6. Vergleich der Algorithmen (Gesamtüberblick)

| Kriterium | FIFO/TRANSFER | MIXOR | CoFlo |
|-----------|---------------|-------|-------|
| **Domäne** | Transportbeton | Transportbeton | Zement (FTL) |
| **Ansatz** | Heuristik (2-stufig) | Heuristik (verbessert) | Heuristik + MIP |
| **FiFo-Regel** | Ja (mit Ausnahmen) | Ja (mit Ausnahmen) | Nein |
| **Mehrschicht** | Nein | Ja | Ja |
| **Pausenplanung** | Nein | Ja (statisch + dynamisch) | Nein (nicht relevant) |
| **Standalone** | Nein | Ja | Ja |
| **Input/Output** | Proprietär | JSON | – |
| **Solver** | Heuristik | Heuristik | SCIP/CBC (MIP) |
| **CPU-Zeit** | Höher | ~7× schneller als TRANSFER | – |
| **Wartungsaufwand** | Hoch | Gering–Mittel | – |

---

## 7. Dispatch-Gruppen und Optimierungstypen (Kontext)

Jede **Dispatch-Gruppe (DG)** hat folgende Optimierungstypen:

| Typ | Bedeutung |
|-----|-----------|
| **RT** | Realtime-Optimierung (laufende Disposition) |
| **OEV0** | Optimierte Endvorschau Typ 0 (Tagesplanung) |
| **OEV1** | Optimierte Endvorschau Typ 1 |
| **OEV2** | Optimierte Endvorschau Typ 2 |
| **PP** | Preplanning (Vorplanung für nächsten Tag) |

**Scheduling-Verhalten:**
- Tserv forkt für jeden Typ einen Prozess → RT, OEV0, OEV1+2, PP laufen **parallel** innerhalb einer DG
- Pro Typ: max. ein Prozess gleichzeitig; weitere Anfragen warten in der Queue
- Jede Queue ist striktes FIFO – keine typübergreifende Priorisierung
- Mehrere RT-Optimierer laufen **nicht** parallel (nur einer gleichzeitig pro DG)

---

## 8. Schlüsselbegriffe und Abkürzungen

| Begriff | Bedeutung |
|---------|-----------|
| DG | Dispatch Group / Dispatch-Gruppe |
| SAS | Signed Assign Sign-On (zugewiesenes Einloggen) |
| MR | Morning Roster (Morgen-Liste der Fahrzeuge am Werk) |
| OS | Optim-Schedule (Optimierungsergebnis-Zeitplan) |
| TS | Tour-Schedule (produktiver Zeitplan) |
| FTL | Full Truck Load (komplette Fahrzeugladung) |
| LAP | Linear Assignment Problem |
| MIP | Mixed-Integer Programming |
| SCIP | Solving Constraint Integer Programs (Open-Source-Solver) |
| CBC | Coin-or Branch and Cut (Open-Source MIP-Solver) |
| FiFo | First-in-First-Out |
| PlantDispatch | Planungseinheit für ein einzelnes Werk (MIXOR) |
| PlantDispatcher | Algorithmus-Komponente für die werksbasierte Zuweisung |
| ftlopt | Framework für FTL-Optimierung; enthält MIXOR als konkreten Ansatz |
| MIXOR | Mixed Operations Research (neue Betonoptimierung) |
| TRANSFER | Älterer FIFO-basierter Algorithmus (Vorgänger von MIXOR) |
| CoFlo | Cement Flow Optimization (Zementoptimierung) |

---

## 9. TODOs / Offene Punkte (aus den Dokumenten)

- MIXOR Realtime-Anbindung: Klärung der Behandlung von nicht-genommenen Pausen
- MIXOR RT: Integration von DigitalFleet (offene Frage aus RT-Anbindung Doku)
- MIXOR: `theLTLOptim` Objekte für FTL-Modell umbenennen (TODO in Code)
- MIXOR RT: A-Time-Berechnung bei vorgeschlagenen Pausen prüfen
- Roadmap-Item: Verbesserung der Transformationen (Delivery-Transformationen für MIXOR)
- CoFlo: Set Partitioning Extension als optionale Erweiterung (Mehraufwand durch MIP)

---

*Erstellt aus den Quelldokumenten des Ordners `MIXOR_Dokumentation/`, `FIFO/`, `CoFlo/` und den Anforderungsdokumenten der Bachelorarbeit.*
