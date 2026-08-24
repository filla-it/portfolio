# Case Study: Performance-Optimierung eines selbstgehosteten GitHub-Actions-Runner-Clusters

**Umfang:** allein umgesetzt, von der Architekturentscheidung bis zum Betrieb  
**Stack:** Kubernetes (k3s), Helm, Actions Runner Controller (ARC), Nutanix HCI, Prometheus, Loki, Docker/DinD, GitHub Actions API

## Ausgangslage

Ein selbstgehostetes Runner-Cluster auf k3s/Nutanix bediente die CI/CD-Workloads mehrerer Repositories. Die Kern-Workflows liefen mit ~150–250 Minuten deutlich langsamer als aus der Compute-Ausstattung zu erwarten war. Die naheliegende Vermutung "CPU oder RAM knapp" war falsch: CPU-Auslastung im Schnitt bei 3–4%, RAM bei ~10%. Das eigentliche Problem lag woanders, und war zu Beginn nicht offensichtlich, weil die Standard-Dashboards keinen offensichtlichen Engpass anzeigten.

## Vorgehen

**Engpass in der richtigen Schicht identifiziert.** Statt an CPU/RAM zu drehen, erst die Metriken auf allen Ebenen (Container, Node, Storage-Fabric) verglichen. Auffällig war die Disk-Latenz: nachts 2–4 ms (gesundes SSD-Verhalten), tagsüber 20–58 ms, auf einem einzelnen Node zeitweise 225 ms. Diese Werte sind typisch für HDD unter Last, nicht für SSD-Cache. Der Nutanix-Storage-Pool hatte reichlich freie SSD-Kapazität (5,2 TiB frei bei 2,3 TiB belegt), das Problem lag also nicht an Kapazität, sondern am Tiering-Verhalten.

**Nutanix-ILM als strukturelle Ursache erkannt.** Nutanix nutzt "Intelligent Lifecycle Management" (ILM), das Blöcke automatisch zwischen SSD- und HDD-Tier verschiebt: Hot Data auf SSD, Cold Data auf HDD. Für CI-Workloads passt diese Heuristik strukturell schlecht: ein Runner schreibt einen Build-Output einmalig, liest ihn ein paarmal und verwirft ihn. ILM stuft solche Daten schnell als "cold" ein und demotiert sie noch während der Job läuft auf HDD, obwohl der Runner sie in Sekunden wieder liest. Damit landet der Hot-Path des Builds auf einer HDD, die sich mehrere gleichzeitig aktive Runner teilen.

**Naheliegende Lösungen bewusst verworfen.** Der erste Impuls "Nutanix Flash Mode aktivieren" (Pinning von vDisks auf SSD-only) ließ sich nicht umsetzen: Nutanix hat das Feature ab AOS 7.0 ersatzlos entfernt und durch automatisches ILM ersetzt. Ebenso verworfen: Compression, Deduplication, Erasure Coding im Storage-Container aktivieren. Alle drei lösen das falsche Problem (Kapazität, nicht Latenz) und würden zusätzlichen I/O-Overhead auf den ohnehin überlasteten Pfad legen. Ein separater SSD-only Storage-Pool per CLI wäre möglich, war aber der letzte Ausweg. Vorher wollte ich prüfen, ob sich das Verhalten schon durch weniger Kontention verbessern lässt.

**A/B-Test statt Bauchgefühl.** Statt sofort die Runner-Anzahl massiv zu ändern, ein kontrollierter Test: `maxRunners` von 50 auf 40 gesenkt, Baseline-Metriken der Workflow-Laufzeiten vorher exakt erfasst (aus der GitHub-Actions-API), dann über mehrere Wochen laufen lassen. Die Reduktion war bewusst moderat, denn zu wenige Runner hätten die Queue-Zeit für Entwickler spürbar verschlechtert, das Ziel war die kleinste Änderung, die überhaupt einen Effekt zeigt.

**Kommunikation an die betroffenen Teams begleitet.** Änderung nicht heimlich ausgerollt, sondern vorher mit Begründung ans Dev-Team kommuniziert (Disk-I/O als Ursache, warum das Absenken der Runner-Zahl trotz freiem CPU/RAM sinnvoll ist). Gleichzeitig zwei alternative Vorschläge zur Zeitplan-Umverteilung angeboten, weil die Engpass-Fenster stark tageszeitabhängig sind (Werktags 08:00–14:00 CEST durch Überlappung mehrerer paralleler Test- und Compilation-Suites).

**Zusätzlicher Storage-Vorfall diagnostiziert und eingeordnet.** Während des A/B-Tests fiel auf einem Runner-Node kurzzeitig der freie Root-Filesystem-Platz auf 3,7 GiB (unter der Kubernetes-Eviction-Schwelle von 10%). Erste Reaktion wäre "sofort emptyDir sizeLimit setzen und Docker-Cache aufräumen" gewesen. Vor der Aktion aber erst in Loki über den vollen 30-Tage-Retentionszeitraum geprüft, ob es überhaupt Pod-Evictions, OOM-Kills oder "no space left"-Fehler in den Container-Logs gab. Ergebnis: keine. Docker BuildKit hat seinen Layer-Cache selbst rechtzeitig geräumt, das Node ist am Folgetag von selbst wieder auf 124 GiB frei gesprungen. Kein Handlungsbedarf, aber die Diagnose ohne Panik-Fix dokumentiert.

**Runner-Ressourcen an die tatsächliche Nutzung angepasst.** Das runner-Container-Limit stand historisch auf 10 GiB Memory, ohne dass die Runner das je ausgenutzt hätten (gemessene Working-Set-Größe deutlich unter 1 GiB pro Pod). Auf den GitHub-Default von 7 GiB abgesenkt, um pro Node mehr Pod-Dichte ohne Overcommit zu ermöglichen. Der DinD-Sidecar blieb bewusst auf 10 GiB, weil dort der Docker-Layer-Cache echten Platz braucht.

**Chart-Upgrade sauber ausgerollt.** Der eigentliche Rollout musste laufende CI-Jobs schonen. Zwei-Phasen-Deploy: erst `minRunners` auf 0 setzen (idle Runner terminieren, laufende bleiben unangetastet, ARC forciert keine Termination auf Config-Change), dann `minRunners` auf 23 zurücksetzen. Nachträglich in der GitHub-API geprüft: keine Cancelled/Failure-Runs im Deploy-Fenster (~4 Minuten). Sauber geleert, kein Job verloren.

## Bemerkenswerte Diagnosearbeit unterwegs

- Ein früherer Analyse-Versuch wurde von einem Zeitstempel-Fehler ausgehebelt: die Deploy-Zeit einmal aus einem Unix-Timestamp berechnet, dabei versehentlich das Jahr um eins verschoben (2025 statt 2026). Dadurch wanderten alle "Pre-Deploy"-Werte in einen leeren Retentionsbereich und die Auswertung klassifizierte alle Daten als "Post-Deploy". Erst der offensichtliche Widerspruch (Werte waren gleich schlecht wie vorher) führte zur Ursache. Konsequenz: Timestamps aus `datetime`-Objekten mit expliziter Timezone konstruieren, nie hartkodierte Unix-Sekunden übernehmen.
- Loki-Queries über den vollen 30-Tage-Retentionszeitraum liefen wiederholt in Timeouts (30s serverseitig, unabhängig von Client-Timeout). Statt das Limit hochzudrehen, in Wochen-Chunks zerlegt und pro Chunk separat abgefragt. Funktionierte auf Anhieb.
- `avg()` in Prometheus über Serien mit NaN-Werten liefert NaN, nicht den Durchschnitt der validen Werte. Ergebnis war erst irritierend leer. Auf `sum() / count()` umgestellt.
- Prometheus `query_range` hat ein implizites Punkte-Limit (~11.000). Über größere Zeiträume mit feinem Step läuft die Abfrage stumm ins Leere. Step auf 3600s erhöht, Auflösung war für Tagesminima ausreichend.

## Auszüge aus der Analyse

Die folgenden Snippets sind generische Nachbauten der tatsächlich verwendeten Muster, keine wörtlichen Zitate aus dem privaten Repository.

**Zwei-Phasen-Deploy, um laufende CI-Jobs beim Config-Rollout nicht zu killen:**

```bash
# Phase 1: idle Runner abbauen (minRunners=0). ARC bricht laufende
# Jobs nicht ab; ephemere Pods leben bis zum Job-Ende weiter.
yq -i '.minRunners = 0' values.yaml
helm upgrade arc-runner-set <chart> -n arc-runners -f values.yaml

# kurz warten, bis idle Pods terminiert sind

# Phase 2: neue Config aktivieren, min-Runner-Pool wieder aufbauen
yq -i '.minRunners = 23' values.yaml
helm upgrade arc-runner-set <chart> -n arc-runners -f values.yaml
```

**Loki-Abfrage in Wochen-Chunks, weil eine Anfrage über den vollen Retentionszeitraum serverseitig in ein Timeout läuft:**

```python
weeks = [
    ("2026-07-07 18:00", "2026-07-14 18:00"),
    ("2026-07-14 18:00", "2026-07-21 18:00"),
    # ... jede Woche einzeln
]
for term in ["evict", "OOMKill", "no space left"]:
    for ws, we in weeks:
        start_ns = int(iso_to_utc(ws).timestamp() * 1e9)
        end_ns   = int(iso_to_utc(we).timestamp() * 1e9)
        query_loki({
            "query": f'{{namespace="arc-runners"}} |= "{term}"',
            "start": start_ns, "end": end_ns,
        })
```

**Verifikation via GitHub-Actions-API: wurden im Deploy-Fenster Jobs gekillt?**

```python
w_start = datetime(2026, 8, 6, 18, 50, tzinfo=timezone.utc)
w_end   = datetime(2026, 8, 6, 19, 10, tzinfo=timezone.utc)
for repo in recently_pushed_repos:
    runs = gh_api(f"/repos/{ORG}/{repo}/actions/runs?per_page=50")
    for r in runs["workflow_runs"]:
        upd = parse_iso(r["updated_at"])
        if w_start <= upd <= w_end and r["conclusion"] in ("cancelled", "failure"):
            print(f"suspect: {repo} {r['name']} {r['conclusion']}")
```

## Ergebnis

Über die vier Kern-Workflows hinweg zwischen 3 und 7% Laufzeitreduktion je nach Workflow, gemessen über 3,5 Wochen Post-Deploy-Betrieb (n=15–17 erfolgreiche Runs pro Workflow), bei unveränderter Hardware und ohne zusätzliche Kosten. Konfigurations-Rollout ohne einen einzigen abgebrochenen Job. Storage-Vorfall auf einem Node ohne Handlungsbedarf, weil die Analyse zeigte, dass das System sich selbst stabilisiert hat. Die Erkenntnis zum Nutanix-ILM-Verhalten dokumentiert, damit sie bei künftigen Storage-Diskussionen (SSD-only Pool, Hardware-Erweiterung) als Argumentationsgrundlage verfügbar ist.

## Gezeigte Fähigkeiten

Kubernetes (Helm-basierte ARC-Konfiguration, Pod-Ressourcen-Tuning, Eviction-Verhalten), Storage-Analyse (Nutanix ILM, Tiering, Latenz- vs. Kapazitäts-Diagnose), Observability (Prometheus TSDB-Queries, Loki-LogQL, Retention-Grenzen), Change-Management (kontrollierter A/B-Test, kommunizierte Rollouts, Zwei-Phasen-Deploy zur Job-Schonung), Root-Cause-Analyse (Engpass in der richtigen Schicht suchen statt an CPU/RAM zu drehen, Selbstkritik bei fehlerhaften Auswertungen), kostenbewusstes Infrastruktur-Design (weniger Runner statt mehr Hardware).

---

[← Zurück zur Übersicht](README.md)
