# Portfolio

Vier Fälle aus dem Aufbau und Betrieb interner Plattform- und CI-Infrastruktur. Jeweils alleine umgesetzt, von der Architekturentscheidung bis zum Betrieb.

Der Schwerpunkt liegt weniger auf dem eingesetzten Werkzeug als auf den Entscheidungen: was gebaut wurde, was bewusst nicht, und was wieder entfernt wurde.

## Fälle

**[Self-Service-Plattform für Entwickler-Workspaces](workspace-platform-case-study.md)**
Aus manuell gepflegten Einzelumgebungen wurde eine Selbstbedienungsplattform: 26 Umgebungen für rund 14 Nutzer, ohne dass Versions- oder Reset-Wünsche über eine Person laufen.
*Docker Compose, Traefik, Python, Node.js/Express, React, XFS-Quotas*

**[Migration einer internen Anwendung auf Kubernetes, Terraform-first](kubernetes-migration-case-study.md)**
Aus einer App-Migration wurde der Aufbau der Ingress-, TLS- und Load-Balancing-Fähigkeit des gesamten Clusters. Zwei Entscheidungen wurden unterwegs revidiert.
*Terraform, Helm, k3s, cert-manager, MetalLB, ACME DNS-01, ArgoCD*

**[CI/Security-Pipeline für eine intern gehostete Docker-Infrastruktur](ci-security-pipeline-case-study.md)**
Erstmals systematisches Container-Vulnerability-Scanning für eine Infrastruktur, die zuvor ungeprüft ausgeliefert wurde. Ein fertiger Scan-Job wurde dabei wieder entfernt.
*GitHub Actions, selbstgehostete Runner, Trivy, Dependency-Governance*

**[Performance-Optimierung eines selbstgehosteten GitHub-Actions-Runner-Clusters](arc-runner-optimization-case-study.md)**
Der vermutete CPU-Engpass war keiner. Die Messung zeigte Disk-I/O, die Konfiguration wurde datenbasiert nachgesteuert.
*Kubernetes, ARC, Prometheus, Loki, Nutanix-Storage, kontrollierter A/B-Rollout*

## Zu den Texten

Firmennamen, Hostnamen, interne Pfade und vergleichbare Details sind bewusst verallgemeinert. Die technischen Abläufe und Entscheidungen sind unverändert.

Bei Recherche und Umsetzung arbeite ich mit KI-Assistenz. Die Entscheidungen in diesen Fällen sind meine, und mehrere gingen gegen den naheliegenden Vorschlag: Der erste CI-Entwurf wurde verworfen, ein fertiger Scan-Job wieder entfernt, die Registry-Entscheidung mittendrin revidiert.
