# Infos für die Bachelorarbeit – OptimScheduler

## 1. Thema & Zielbild

**Thema:** Implementierung und Konzeptionierung eines Webservice zum Scheduling der SYNCROTESS BM Transportoptimierung.

**Betreuer:** INFORM GmbH (Institut für Operations Research und Management GmbH)

**Kernproblem:** Optimierer-Prozesse laufen aktuell als Fork (neuer Prozess) auf demselben Host-System wie der Tserv. Sie sind durch die Hardware des Host-Systems limitiert, können nicht parallel ausgeführt werden und sind nicht über eine standardisierte Schnittstelle ansprechbar.

**Ziel des OptimSchedulers:**
- Optimierungsprozesse vom Tserv entkoppeln und über REST-Schnittstelle steuern
- Mehrere Optimierungsprozesse parallel ausführen
- Eingehende Anfragen über Warteschlange koordinieren
- Laufzeitgrenzen und Arbeitsspeicherauslastung berücksichtigen
- Geeignete Scheduling-Strategie durch Vergleich von Algorithmen ermitteln
- Die Implementierung dient der Evaluierung (keine Produktivnutzung), Ziel ist fundierte Entscheidungsgrundlage

---

## 2. Ist-Analyse: Tserv & Optimierer

### Tserv
- **Backend des SYNCROTESS-Produkts für das Transportmanagement**
- Enthält die Businesslogik und führt die Optimierer intern aus
- GitLab: `GB20 / Road / Tserv / road`
- Optimierer-Prozesse werden als **Fork** gestartet (neuer Prozess auf demselben Host-System)
- Dadurch sind die Prozesse durch die Hardware des Host-Systems limitiert – diese kann nur eingeschränkt oder umständlich angepasst werden
- Beim Auschecken baut er die Optimierer mit `make`
- Beim Übersetzen liegt in `$INSTROOT/bin` das `ftlopt` executable
- Tserv ruft Shell-Skripte auf (doFtlopt.sh.tmpl)
- ftlOpt-Skripte wären standalone ausführbar, werden aber ausschließlich über den Tserv gestartet

### Optimierer (bekannte Typen)
| Optimierer | Repo | Zweck |
|---|---|---|
| `ftlopt` | `GB20 / Road / Tserv / ftlopt` | Hauptoptimierung |
| `mixor` | – | Weiterer Optimierer |
| `coflo` | `GB20 / Road / Tserv / coflo` | Zementoptimierung |

### Relevante Pfade & Kommandos
```bash
# Optimierungsaufruf
ftlopt -i ftlopt1.json -w /nethome/tziarnet/tmp/ -s ftlopt1.sol.json

# Aufruf der ausgewählten Optimierung im Tserv
$SRCROOT/common/optimlib/run_optim.cc

# Shell-Skript-Template für ftlopt/MIXOR
$SRCROOT/road/script/doFtlopt.sh.tmpl
```

### Dispatch-Gruppen und Optimierungstypen
- Optimierungsläufe sind nach **Dispatch-Gruppen** organisiert (z.B. Planungsregion oder Werk)
- Jede Dispatch-Gruppe hat folgende Optimierungstypen: **RT, OEV0, OEV1, OEV2, PP**
- Tserv erstellt pro Typ einen eigenen Fork → RT, OEV0, OEV1&2 (gemeinsam), PP laufen **innerhalb einer Dispatch-Gruppe parallel**
- Pro Typ läuft höchstens **ein Prozess gleichzeitig**; weitere Anfragen warten in der Warteschlange
- Jede Warteschlange ist strikt **FIFO** – keine Priorisierung innerhalb oder zwischen den Queues
- In der Praxis wird ein Slot nicht mehrfach gleichzeitig angefragt, da der Aufrufer warten muss

### Probleme des Ist-Zustands
- **Starre, hartcodierte Parallelitätsstruktur** – Parallelität ist fest an Dispatch-Gruppen-Typen gebunden, nicht konfigurierbar
- **Kein typenübergreifendes Scheduling** – RT, OEV und PP-Queues sind voneinander isoliert, keine übergreifende Priorisierung möglich
- **Nur FIFO innerhalb der Queues** – keine Priorisierung dringender Anfragen innerhalb eines Typs
- **Änderungen erfordern Tserv-Quellcode-Änderungen** – neue Typen oder andere Parallelitätsgrade nicht konfigurierbar
- Optimierer-Prozesse laufen als Fork auf dem Host → Hardware-Limitierung, geringe Skalierbarkeit
- Optimierer ausschließlich über den Tserv ansprechbar
- Keine Laufzeitkontrolle / Ressourcenüberwachung

---

## 3. Funktionale Anforderungen

| Eingabe / Ereignis | Erwartetes Verhalten / Ausgabe |
|---|---|
| Optimiererauswahl & Payload | Optimiererergebnis wird zurückgegeben |
| Optimierer läuft & Zweiter wird gestartet | Results werden parallel verarbeitet |
| Scheduler ist ausgelastet & Weiterer Optimierer wird gestartet | Anfrage kommt in Warteschlange |
| Warteschlange ist voll & Optimierer wird gestartet | Fehlermeldung |
| Optimierer fertig & Warteschlange gefüllt | Nächste Anfrage aus der Warteschlange wird verarbeitet |
| Optimiererpayload bei Anfrage unvollständig | Fehlermeldung & Info |
| Einzelner Optimierer ist fertig | Laufzeit wird mit zurückgeschickt |
| Arbeitsspeicher stößt an die Belastungsgrenze | Kein weiterer Optimierer kann gestartet werden |
| Optimierer wird angesprochen der nicht existiert | Entsprechende Fehlermeldung |
| Optimierung dauert zu lange | Entsprechende Fehlermeldung (Timeout) |
| Optimierung ohne Authentifizierung | Entsprechende Fehlermeldung |
| Erster Prozess läuft, zweiter Prozess überschreibt den ersten | Erster Prozess wird gestoppt, zweiter läuft durch und gibt Result zurück |

---

## 4. Technischer Aufbau & Architektur

### Gesamtarchitektur

```
Client
  ↓  REST (HTTPS/JSON)
Spring Boot (OptimScheduler – Java, Maven)
  ↓
Scheduler + Queue + Strategies
  ↓
Kubernetes API
  ↓
Pods (Optimierer als Subprozesse / Container)
```

### Rolle des OptimSchedulers
Der OptimScheduler ist das zentrale Steuermodul zwischen:
- Eingehenden Requests (REST)
- Warteschlange (Job-Queue)
- Laufenden Optimierer-Instanzen (Prozesse oder Pods)
- Systemressourcen (RAM, CPU)

Er ist **zustandsbehaftet** – die Optimierer selbst sind zustandslos.

### Kernaufgaben des Schedulers

| Aufgabe | Erklärung |
|---|---|
| Optimiererauswahl | Mapping `optimizer_type` → ausführbares Kommando (ftlopt, mixor, …) |
| Job-Start | Prozess oder Container starten |
| Parallelität | Mehrere Jobs gleichzeitig zulassen |
| Queue-Verwaltung | Blockieren oder Ablehnen bei Überlast |
| Timeout-Handling | Abbruch bei zu langer Laufzeit (max. 2 Minuten) |
| Ressourcenüberwachung | RAM-/CPU-Limits |
| Fehlerbehandlung | Konsistente Fehlercodes |

### Designentscheidungen
- Jeder Optimiererlauf ist ein eigener **Kubernetes Pod**
- **Async Execution** – Server blockiert nicht auf Ergebnisse
- Spring Boot Application läuft in eigenem **Docker Container**
- **REST-Schnittstelle** (flexibel, skalierbar; dateibasierte Kommunikation scheidet im verteilten System aus)
- **Non-preemptive Priority Queue** (RT-basiert, da Burst Times unbekannt)

### Interner konzeptioneller Aufbau
```
+-----------------------+
| REST API              |
+-----------------------+
            |
            v
+-----------------------+
| OptimScheduler        |
| - Job-Queue           |
| - Worker-Management   |
| - Resource Monitor    |
+-----------------------+
     |            |
     v            v
 Optimizer A   Optimizer B
 (ftlopt)      (mixor)
```

> **Wichtig:** Der Scheduler kennt den Tserv – der Optimierer kennt den Tserv **nicht**.

---

## 5. Technologie-Stack

| Komponente | Technologie |
|---|---|
| Backend / Scheduler | Java, Spring Boot |
| Build-Tool | Maven |
| Containerisierung | Docker |
| Orchestrierung | Kubernetes (konzeptionell) |
| Schnittstelle | REST / HTTPS / JSON |
| Betriebssystem | Linux |
| Optimierer-Binaries | ftlopt, mixor (C++ mit Boost-Bibliotheken) |
| Build der Optimierer | `make` (Makefile im Repo) |

---

## 6. REST-Schnittstelle

### Endpunkte

**Job starten:**
```
POST /optimize
Authorization: Bearer <token>

Request:
{
  "optimizer": "ftlopt",
  "payload": "input.json",
  "timeout": 300
}

Response:
{
  "jobId": "abc123",
  "status": "queued"
}
```

**Job-Status abfragen (Polling):**
```
GET /optimize/{jobId}

Response:
{
  "status": "finished",
  "duration": 42.3,
  "result": "output.json"
}
```

**Weitere Endpunkte (geplant):**
- Statusabfrage eines Optimiererlaufs (wichtig für Polling)
- Progress Updates
- Replace / Update (neuere Anfrage ersetzt laufende)
- Cancellation / Abbruch

### Fehlercodes

| Fall | HTTP-Code |
|---|---|
| Keine Authentifizierung | 401 |
| Unbekannter Optimierer | 400 |
| Queue voll | 429 |
| Timeout | 504 |
| Payload unvollständig | 422 |
| RAM-Limit erreicht | 503 |

### Job-Zustände
- `queued` – wartet in der Queue
- `running` – wird aktuell ausgeführt
- `finished` – erfolgreich abgeschlossen
- `failed` – fehlgeschlagen
- `timeout` – abgebrochen wegen Zeitüberschreitung

---

## 7. Scheduling-Algorithmen

### Killer-Kriterien (Ausschlusskriterien)
1. **Burst Times sind nicht bekannt** – Optimierungszeiten variieren stark und sind vorab nicht abschätzbar → Algorithmen die Burst Time voraussetzen (SJF, LSF, HRRN, SRTF) **scheiden aus**
2. **Non-preemptive bevorzugt** – Das Unterbrechen und Fortführen von laufenden Optimierungsprozessen ist nicht sinnvoll/möglich → Preemptive Algorithmen grundsätzlich **nicht geeignet**
3. **Ausnahme:** Expliziter Cancel-Endpunkt für den Spezialfall (z.B. LKW 1 fällt aus → Optimierung 1 darf abgebrochen werden)

### Betrachtete Algorithmen

| Algorithmus | Burst Time nötig? | Preemptiv? | Geeignet? | Begründung |
|---|---|---|---|---|
| **FIFO (First In First Out)** | Nein | Nein | ✅ Basis | Einfachste Strategie, guter Vergleichsbaseline |
| **Non-Preemptive Priority Scheduling** | Nein | Nein | ✅ **Gewählt** | RT-basierte Priorität, läuft bis Ende durch |
| **Preemptive Priority Scheduling** | Nein | Ja | ⚠️ Eingeschränkt | Unterbrechung möglich, aber problematisch |
| **Deadline Monotonic Scheduling (DMS)** | Nein | Ja | ❌ | Preemptiv, feste Prioritäten (statisch) |
| **Earliest Deadline First (EDF)** | Nein | Ja | ❌ | Preemptiv, dynamische Prioritäten |
| **Least Slack Time First (LSF)** | Ja | Ja | ❌ | Braucht Burst Time + preemptiv |
| **MultiLevel Feedback Queue (MLFQ)** | Nein | Bedingt | ⚠️ Komplex | Dynamische Prioritäten, Starvation-Prävention |
| **MultiLevel Queue (MLQ)** | Nein | Bedingt | ⚠️ Komplex | Wie MLFQ, aber kein Verschieben der Prozesse |
| **HRRN (Highest Response Ratio Next)** | Ja | Nein | ❌ | Braucht Burst Time |
| **Shortest Remaining Time First (SRTF)** | Ja | Ja | ❌ | Braucht Burst Time + preemptiv |

### Gewählter Algorithmus: Non-Preemptive Priority Scheduling
- Braucht keine Burst Time
- CPU wird einem laufenden Prozess nicht weggenommen
- Auch wenn ein Prozess höherer Priorität ankommt, muss er warten
- Aktuell laufender Prozess wird vollständig beendet
- **Kernaussage:** „Einmal gestartet, läuft der Prozess bis zum Ende."
- Quelle: https://www.geeksforgeeks.org/operating-systems/priority-scheduling-in-operating-system/

### Kriterien für die Priorität (mit Timm abzustimmen)
- RT (Response Time) ist das wichtigste Kriterium
- Priority Queue definitiv
- Weitere Kriterien nach Performanz-Tests
- `maxWaitingTime`? `maxTime`? Wie steigt Priorität mit der Zeit?

---

## 8. Parallelität & Grenzen

- Mehrere Jobs sollen gleichzeitig laufen können
- **Grenzen noch offen:** Auslastung der Prozessoren? Fest definierte Anzahl paralleler Prozesse?
- Optimierungsdauer: **max. 2 Minuten** (Requirement von Timm)
- Queue-Größe: Wie wird die Grenze festgelegt? (Auslastungsbasiert?)

---

## 9. Docker & Kubernetes

### Docker (Containerisierung der Optimierer)

**Idee:** Jeder Optimierer läuft in einem eigenen Container.

Container enthält:
- `ftlopt` binary
- Benötigte Libraries (z.B. Boost)
- Startskript (`doFtlopt.sh`)

```bash
docker run \
  -v /data/job123:/work \
  ftlopt-image \
  ftlopt -i input.json -s output.json
```

**Vorteile:**
- Saubere Ressourcengrenzen (RAM-Limit, CPU-Limit)
- Reproduzierbarkeit (gleiche Umgebung, gleiche Libraries)
- Sicherheit (kein direkter Zugriff auf Host)
- Idealer Übergang zu Kubernetes

### Kubernetes (konzeptionell / Ausblick)

| Anforderung | Kubernetes-Konzept |
|---|---|
| Parallelität | Pods |
| Warteschlange | Job + Controller |
| RAM-Grenzen | Resource Limits |
| Timeout | `activeDeadlineSeconds` |
| Skalierung | Horizontal Pod Autoscaler |

**Zielbild mit Kubernetes:**
```
Tserv
  |  REST
  v
OptimScheduler (Service im eigenen Docker Container)
  |  creates
  v
K8s Job --> Pod (Optimierer-Container)
```

> Der OptimScheduler **bleibt** – Kubernetes übernimmt nur die Ausführung.

### Architekturvariante: Prozessbasiert (ohne Docker)
Alternativ: Optimierer per `fork/exec` oder `std::system()` direkt starten.
- Überwachung via Exit Codes, Laufzeit, Speicherverbrauch (`/proc` unter Linux)
- ✅ Einfach, kein Docker-Zwang, passt zu bestehendem Tserv-Code
- ❌ Schlechtere Isolation, skaliert schlecht auf mehrere Maschinen

---

## 10. Scope des Prototyps

### Im Scope (wird umgesetzt)
- OptimScheduler als eigenständiger Prozess
- REST-API (minimaler Umfang)
- Job-Queue und Job-Status
- Starten eines Optimierers (z.B. ftlopt)
- Timeout- und Fehlerbehandlung
- Rückgabe von Laufzeit und Status
- Whitebox-Tests (Memory Pressure, Off-Limit, Strategie-Tests)

### Out of Scope (nur konzeptionell)
- Kubernetes-Cluster (vollständige Umsetzung)
- Mehrknotenbetrieb
- Persistente Datenbank
- Hochverfügbarkeit / Security Hardening

---

## 11. Implementierungsphasen

| Phase | Ziel |
|---|---|
| **Phase 1** – Architektur & Grundlagen | Komponenten definieren, Schnittstellen, Zustandsdiagramm → direkt Kapitel 4 der Arbeit |
| **Phase 2** – Grundlegender OptimScheduler | HTTP-Server, `POST /optimize`, `GET /optimize/{jobId}`, FIFO-Queue, Prozessstart |
| **Phase 3** – Parallelität & Queue-Logik | Max. Paralleljobs, Queue bei Auslastung, serielles Nachrücken |
| **Phase 4** – Ressourcen & Timeout | Laufzeitüberwachung, RAM-Monitoring, Rückgabe von Laufzeit & Abbruchgrund |
| **Phase 5** – Docker (optional) | Container pro Optimierer, Input/Output per Volume |
| **Phase 6** – Kubernetes (rein konzeptionell) | Job → K8s Job, Optimierer → Pod, Ausblick |

---

## 12. Testing

- **Whitebox-Tests** der Scheduling-Strategie
- Szenarien: Memory Pressure, Queue Off-Limit, Timeouts
- Messdaten tracken (Laufzeiten, Wartezeiten, Queue-Länge)
- Testdaten beschaffen/erstellen (Standalone-Optimierer)
- Scheduling-Algorithmen auf Basis von Messdaten vergleichen und bewerten

---

## 13. Offene Punkte / TODOs

- [ ] Kriterien für Scheduling-Algorithmen mit Timm festlegen (Priority, `maxWaitingTime`, `maxTime`, Prioritätssteigerung über Zeit?)
- [ ] Grenzen der Queue festlegen (auslastungsbasiert?)
- [ ] Standalone-Binaries (ftlopt etc.) besorgen (Makefile im Repo ausführen, Boost-Bibliotheken benötigt)
- [ ] Schnittstelle implementieren
- [ ] Messdaten-Tracking implementieren
- [ ] Testdaten erstellen / besorgen
- [ ] Erweiterungsfall klären: Mehrere Kunden nutzen dieselbe Schnittstelle → Ressourcentrennung?

---

## 14. Kapitelstruktur der Bachelorarbeit

```
1. Einleitung & Motivation
2. Ist-Analyse (Tserv & Optimierer)
3. Anforderungsanalyse
   3.1 Gegebene Anforderungen
   3.2 Anwendungsfälle
   3.3 Scheduling-Algorithmen (Überblick)
   3.4 Killer-Kriterien & Eingrenzung
4. Konzeptionierung & Architektur des OptimSchedulers
   4.1 Architekturdiagramm
   4.2 REST-Spezifikation
   4.3 Zustandsdiagramm
   4.4 Entkopplung vom Tserv
   4.5 Scheduling-Strategie (detailliert)
   4.6 Docker-Containerisierung
   4.7 Kubernetes (konzeptionell)
5. Implementierung
   5.1 Implementierungsentscheidungen
   5.2 Probleme & Lösungen
6. Evaluation
   6.1 Testaufbau & Messdaten
   6.2 Abdeckung der Anforderungen
   6.3 Vergleich der Scheduling-Strategien
7. Diskussion & Ausblick
   7.1 Stärken & Schwächen des Ansatzes
   7.2 Warum Kubernetes sinnvoll wäre
   7.3 Mögliche Weiterentwicklung
   7.4 Erweiterungsfall: Multi-Tenant
8. Fazit
```

---

## 15. Quellen & Referenzen

- Non-/Preemptive Priority Scheduling: https://www.geeksforgeeks.org/operating-systems/priority-scheduling-in-operating-system/
- DMS / EDF / LSF: https://www.informatik.hu-berlin.de/de/forschung/gebiete/rok/teaching/WS/EMES/Slides/03-priority_scheduling.pdf
- MLFQ: https://www.geeksforgeeks.org/operating-systems/multilevel-feedback-queue-scheduling-mlfq-cpu-scheduling/
- MLFQ (OSTEP): https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-mlfq.pdf
- HRRN: https://www.geeksforgeeks.org/operating-systems/highest-response-ratio-next-hrrn-cpu-scheduling/
- SRTF: https://www.geeksforgeeks.org/dsa/shortest-remaining-time-first-preemptive-sjf-scheduling-algorithm/
- GitLab OptimScheduler: https://gitlab.com/evotess/playground/optimscheduler
