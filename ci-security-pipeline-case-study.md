# Case Study: CI/Security-Pipeline für eine intern gehostete Docker-Infrastruktur

**Umfang:** allein umgesetzt, von der Architekturentscheidung bis zum Betrieb  
**Stack:** GitHub Actions, Kubernetes (k3s, selbstgehostete Actions-Runner), Docker/Buildx, Trivy, Dependabot, Bash/jq

## Ausgangslage

Ein internes Projekt baut und verwaltet Docker-Images für Test-/Demo-Umgebungen (Basis-Images, Anwendungs-Images, IDE-Images) sowie zugehörige Infrastruktur (Reverse Proxy, Datenbanken, Admin-Tools). Bisher wurden Images ausschließlich manuell auf dem produktiven Server gebaut und getestet, es gab keine CI. Der naheliegende erste Impuls "dafür brauchen wir GitHub Actions" hatte zwei Probleme: Das Budget für GitHub-gehostete Runner hätte bei Build-Größe und Build-Häufigkeit nicht gereicht, und unklar war, ob eine CI-Pipeline hier überhaupt echten Mehrwert liefern würde oder nur vorhandene manuelle Prozesse verdoppelt.

## Vorgehen

**Bestehende Infrastruktur genutzt statt neu gebaut.** Es existierte bereits ein selbstgehostetes GitHub-Actions-Runner-Cluster (Kubernetes-basiert, mit Autoscaling), allerdings für eine andere GitHub-Organisation registriert. Self-hosted Runner sind technisch fest an eine Org/ein Repo gebunden; das bestehende Cluster konnte das neue Projekt grundsätzlich nicht bedienen, egal wie die Workflow-Datei konfiguriert wird. Statt neuer Hardware wurde auf derselben Maschine eine zweite, unabhängige Runner-Registrierung eingerichtet. Für die Authentifizierung bewusst gegen einen klassischen Personal Access Token entschieden und für eine GitHub App: PATs laufen ab oder erzwingen eine Rotations-Policy und sind an eine Person gebunden: Bricht das unbemerkt ab, fällt CI ohne Vorwarnung aus. Eine GitHub App nutzt kurzlebige, automatisch erneuerte Zugriffstokens und ist nicht an eine Person gekoppelt.

**Design mehrfach unter kritischer Prüfung korrigiert.** Der erste Entwurf spiegelte einfach das bestehende manuelle Build-Skript in CI. Entscheidende Rückfrage: *"Wozu genau? Das Team testet ja eh schon manuell vor jedem Merge."* Ein reiner Build-Smoke-Test hätte nichts abgefangen, was nicht ohnehin schon geprüft wurde, also wurde er umgewidmet: Der Build-Schritt liefert jetzt ein Image, das mit einem Vulnerability-Scanner (Trivy) auf bekannte CVEs geprüft wird, etwas, das vorher niemand systematisch gemacht hat. Beim Nachfragen "deckt das wirklich alles ab?" fiel außerdem auf, dass mehrere Drittanbieter-Images (Datenbank, Reverse Proxy, Admin-Tool) aus Docker Compose gar nicht geprüft wurden. Ergänzt, mit Versionen, die zur Laufzeit aus der bestehenden Projekt-Konfiguration gelesen werden statt ein zweites Mal fest im Workflow zu stehen. Umgekehrt wurde ein bereits fertig gebauter Scan-Job später wieder entfernt, weil er ein Image doppelt zu einem schon geprüften Basis-Image prüfte und die einzige Handlungsoption ohnehin unabhängig vom Scan-Ergebnis immer dieselbe war.

**Alarmierung nachgerüstet statt "Fire and forget".** Ein Scan, dessen Ergebnisse nur in einer Job-Zusammenfassung stehen, die niemand liest, bringt wenig. Ergänzt wurde eine Aggregation: Jeder Scan-Job legt sein Ergebnis als Artefakt ab, ein abschließender Job sammelt alle ein und öffnet, nur bei tatsächlichen kritischen Funden, ein einziges GitHub-Issue pro Lauf mit allen betroffenen Images und CVEs, statt fünfzehn einzelne oder gar keine Benachrichtigung.

**Dependency-Hygiene ohne blindes Durchwinken.** Beim Aktivieren automatisierter Abhängigkeits-Updates zeigte sich ein jahrealter Bestand offener PRs. Ursache: ein längst vollzogener Paketmanager-Wechsel, die PRs zielten auf eine Datei, die es gar nicht mehr gab. Geschlossen statt gemergt. Bei aktuellen Update-Vorschlägen deckte ein neu gebauter Build-Check zwei echte Probleme auf: zwei einzeln vorgeschlagene Updates gehörten wegen gekoppelter Versionsanforderungen zusammen (der kombinierte Vorschlag war der richtige), und ein Update auf eine neue Major-Version brach den Build durch eine echte Breaking Change, bewusst nicht gemergt oder forciert, sondern als eigene Migrationsaufgabe zurückgestellt.

**Absicherung gegen Supply-Chain-Risiken nachgezogen.** Alle verwendeten Third-Party-Actions wurden von beweglichen Versions-Tags auf exakte Commit-SHAs umgestellt (ein Tag kann nachträglich umgebogen werden, ein Commit-Hash nicht), und beide Workflows bekamen explizite Least-Privilege-Permissions statt sich auf die Repo-Default-Rechte des Workflow-Tokens zu verlassen.

## Bemerkenswerte Diagnosearbeit unterwegs

- Die Cross-Org-Bindung von self-hosted Runnern als Plattform-Grenze identifiziert, nicht als lösbares Konfigurationsproblem. Das verhinderte einen aufwändigen Fehlversuch, das per Workflow-Label umgehen zu wollen.
- Ein isolierter Docker-Buildx-Treiber, der ein gerade selbst gebautes, nur lokal geladenes Image nicht wiederfand und stattdessen versuchte, es aus einer öffentlichen Registry zu ziehen (wo es nicht existiert). Aus der rohen Fehlermeldung diagnostiziert, gezielt nur für die zwei betroffenen Jobs auf den einfacheren Standard-Treiber zurückgewechselt statt eines globalen Workarounds.
- Eine falsche Annahme über die Sortierreihenfolge einer Versions-Konfigurationsdatei (angenommen "letzte Zeile = neueste", tatsächlich umgekehrt) erst durch Log-Analyse des ersten echten Testlaufs gefunden, nicht durch Code-Review vorher erkannt.
- Eine nicht existierende Ausdrucksfunktion, die den gesamten Workflow unbrauchbar gemacht hatte, über die tatsächliche API-Fehlermeldung aufgespürt statt vermutet.

## Auszüge aus dem CI-Code

Die folgenden Snippets sind generische Nachbauten der tatsächlich verwendeten Muster, keine wörtlichen Zitate aus dem privaten Repository. Struktur und Logik entsprechen dem echten Vorgehen.

**Versionen zur Laufzeit aus der Projekt-Konfiguration lesen statt im Workflow zu duplizieren:**

```yaml
- id: parse
  run: |
    # Config-Datei ist neueste-zuerst sortiert, erst durch einen
    # Fehlschlag im ersten echten Lauf aufgefallen, dass "letzte Zeile"
    # die falsche Annahme war.
    LATEST=$(grep -Ev '^\s*#|^\s*$' versions.txt | head -1)
    echo "version=$LATEST" >> "$GITHUB_OUTPUT"
```

**Der Buildx-Treiber-Fallstrick: ein isolierter Builder sieht kein Image, das ein vorheriger Schritt nur lokal geladen hat:**

```yaml
# docker-container-Treiber (Standard) kann kein Image sehen, das ein
# vorheriger Build-Schritt per `load: true` nur in den lokalen Daemon
# geladen hat. Er versucht stattdessen, es aus einer Registry zu
# pullen. Der "docker"-Treiber teilt sich den Daemon direkt.
- uses: docker/setup-buildx-action@<pinned-sha>
  with:
    driver: docker
```

**Scatter-Gather: mehrere parallele Scan-Jobs melden über Artefakte an einen einzigen Aggregations-Job, der nur bei echtem Fund aktiv wird:**

```yaml
report-critical-findings:
  needs: [scan-images, scan-external-images]
  if: always()
  permissions:
    contents: read
    issues: write
  steps:
    - uses: actions/download-artifact@<pinned-sha>
      with:
        pattern: scan-result-*
        path: results
    - env:
        GH_TOKEN: ${{ github.token }}
      run: |
        total=0
        for f in results/*/result.json; do
          count=$(jq '[.findings[]? | select(.severity=="CRITICAL")] | length' "$f")
          total=$((total + count))
        done
        [ "$total" -eq 0 ] && { echo "Nothing to report"; exit 0; }
        gh issue create \
          --title "Security scan: $total CRITICAL finding(s)" \
          --body-file summary.md
```

## Ergebnis

Automatisierter Sicherheits-Scan über alle selbst gebauten und alle extern bezogenen Images hinweg, PR-Build-Check für eine Komponente, die vorher gar nicht getestet wurde, automatisierte Abhängigkeitspflege mit sinnvollen Leitplanken (nicht blindes Durchwinken), und Supply-Chain-Härtung, alles auf bestehender Infrastruktur, ohne zusätzliche Kosten für GitHub-gehostete Runner. Über mehrere Testläufe verifiziert, inklusive der beiden oben beschriebenen Fehlerfälle, die erst dabei sichtbar wurden.

## Gezeigte Fähigkeiten

GitHub Actions (selbstgehostete Runner, Matrix-/Scatter-Gather-Patterns, artefaktbasierte Job-Koordination), Kubernetes (Helm-basierte Runner-Registrierung, GitHub-App- vs. PAT-Credential-Design), Docker/Buildx (Cache-Backends, Treiber-Fallstricke), Container-Sicherheit (Trivy, CVE-Triage), Dependency-Management (Dependabot-Konfiguration, Peer-Dependency-Konflikte erkennen), Supply-Chain-Härtung (SHA-Pinning, Least-Privilege-Permissions), kostenbewusstes Infrastruktur-Design.

---

[← Zurück zur Übersicht](README.md)
