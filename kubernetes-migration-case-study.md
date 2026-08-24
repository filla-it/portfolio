# Case Study: Migration einer internen Anwendung auf Kubernetes, Terraform-first

**Umfang:** allein umgesetzt, von der Architekturentscheidung bis zum Betrieb  
**Stack:** Terraform, Helm, Kubernetes (k3s), cert-manager, MetalLB, Docker, ArgoCD, Cloudflare

## Ausgangslage

Ein internes Tool zur Team-Kapazitätsplanung lief auf einer einzelnen VM per Docker Compose. Ziel: der Umzug auf den bestehenden k3s-Cluster (`it-cluster`), vollständig als Code über Terraform und Helm verwaltet, mit klarem Weg zu CI/CD, ohne dabei die bestehenden, unabhängigen Produktiv-Workloads auf demselben Cluster zu stören.

Aus "eine App deployen" wurde dabei der komplette Aufbau der Ingress- und TLS-Fähigkeit für den ganzen Cluster von Grund auf, plus echte Root-Cause-Analyse an Infrastruktur, die seit Jahren niemand angefasst hatte.

## Vorgehen

**Infrastructure as Code, nach Zuständigkeit getrennt.** Zwei Terraform-Workspaces: einer für die App selbst (Namespace, Secrets, `helm_release`), einer für Cluster-weite Fähigkeiten (Ingress-Controller, `cert-manager`, Load Balancing), damit die Plattform-Arbeit jeder künftigen App-Migration zugutekommt, nicht nur dieser einen. Der Helm-Chart wurde aus einer kopierten Vorlage aufgebaut und bewusst aufgeräumt: der unpassende Auto-Update-Controller Keel entfernt, der genau die Art Konfigurations-Drift verursacht hätte, die im Cluster an anderer Stelle bereits real aufgetreten war; ein Helm-templated Weg für Registry-Zugangsdaten entfernt, der ein Klartext-Secret in Git geschrieben hätte; `helm.sh/resource-policy: keep` für das Datenbank-Volume ergänzt, nachdem ein reales Datenverlust-Szenario bei einem Terraform-"tainted resource"-Neuaufbau reproduziert wurde.

**Registry-Entscheidung bewusst mittendrin revidiert.** Begonnen wurde mit einer selbst gehosteten Harbor-Registry (volle Kontrolle, keine Abrechnung bei Dritten). Diese wurde erfolgreich zum Laufen gebracht, inklusive Fix eines seit einem Jahr unbemerkt abgelaufenen Zertifikats und einer Registry-Endpunkt-Fehlkonfiguration, die den gesamten Cluster am Image-Pull gehindert hätte. Dabei wurde klar: Harbor dient in der Organisation primär als CI-Build-Cache für einen *anderen* Cluster, nie für Cross-Cluster-Zugriff vorgesehen. Kurs korrigiert zu einem privaten Docker-Hub-Repository, das der Ziel-Cluster ohnehin schon ohne jegliche Zertifikats-/DNS-Arbeit erreichen konnte: ein Beispiel dafür, das "Warum" hinter bestehender Infrastruktur zu hinterfragen, bevor man weiter darauf aufbaut.

**Fehlende Plattform-Fähigkeit aufgebaut statt Einzelfall-Workaround.** Der Ziel-Cluster hatte weder Ingress-Controller noch `cert-manager` noch einen Load Balancer. Für den fehlenden Ingress-Controller fand sich eine konkrete Ursache: ein Bootstrap-Flag hatte den eingebauten Ingress-Controller des Clusters seit dem allerersten Tag stillschweigend deaktiviert, mit einem seither unbemerkten Fehler in den Logs. `cert-manager` und ein Load Balancer waren dagegen schlicht nie nachgerüstet worden. Statt einer manuellen Einzellösung wurden die fehlenden Bausteine richtig aufgebaut: `ingress-nginx`, `cert-manager` und **MetalLB** (Layer-2-Modus), damit jeder künftige Service einen echten, ausfallsicheren, DNS-adressierbaren Endpunkt bekommt, unabhängig davon, welcher einzelne Node gerade zufällig den Pod hält.

**TLS ohne die zwei naheliegenden, aber falschen Abkürzungen.** Eine private interne CA hätte funktioniert, hätte aber von jedem Nutzer einen manuellen Root-Zertifikat-Import verlangt, bei mehreren Rechnern ohne zentrale Verteilung nicht praktikabel. Automatische HTTP-01-Validierung (der einfache Standardweg) schied aus, weil der Dienst bewusst nicht aus dem öffentlichen Internet erreichbar ist, sondern nur intern per VPN. Gelöst über eine DNS-01-Challenge via Cloudflare: ein echtes, überall vertrauenswürdiges Let's-Encrypt-Zertifikat, ausgestellt ohne den Dienst selbst dem Internet auszusetzen. Die bewusste Internal-only-Zugriffsentscheidung bleibt so erhalten, trotzdem vertraut jeder Browser dem Zertifikat sofort.

**GitOps-bewusst statt gegen GitOps ankämpfend.** Zwei direkte Fixes an geteilter Infrastruktur (ein abgelaufenes Zertifikat, ein kaputter Registry-Endpunkt), zunächst per `kubectl` angewendet, wurden Minuten später kommentarlos zurückgesetzt: ArgoCDs Self-Healing-Sync stellte den Cluster wieder auf einen veralteten Git-Stand zurück. Nach Identifikation beide Fixes stattdessen durch das eigentliche Quell-Repository geleitet, statt gegen die Reconciliation-Schleife anzukämpfen.

## Bemerkenswerte Diagnosearbeit unterwegs

- Eine undokumentierte, bis dahin unbekannte Netzwerkarchitektur (dediziertes VRRP- + nginx-stream-Load-Balancer-Paar vor dem Kubernetes-API-Server) durch direkte Inspektion rekonstruiert, um einen neuen Load-Balancer-IP-Bereich korrekt zu platzieren, ohne mit bestehender Infrastruktur zu kollidieren.
- Eine veraltete DNS-Annahme über drei getrennte Systeme hinweg nachverfolgt (Firewall-DHCP-Konfiguration, Cloud-IPAM, VPN-Client-Einstellung), um einen Zertifikats-SAN-Eintrag zu erklären, der nie tatsächlich funktioniert hatte.
- Sicherheitsrelevante Funde in bestehenden Infrastruktur-Repos während der Arbeit nach tatsächlicher Gefährdung eingeordnet und für weitere Arbeiten dokumentiert.

## Auszüge aus dem Terraform-Code

Domain, IP-Bereiche und Firmenbezüge unten sind anonymisiert. Struktur und Logik entsprechen dem tatsächlich verwendeten Code.

**Ingress-Controller und `cert-manager`, per Helm über Terraform:**

```hcl
resource "helm_release" "ingress_nginx" {
  name             = "ingress-nginx"
  repository       = "https://kubernetes.github.io/ingress-nginx"
  chart            = "ingress-nginx"
  namespace        = "ingress-nginx"
  create_namespace = true

  set {
    name  = "controller.service.type"
    value = "LoadBalancer"
  }
  set {
    name  = "controller.ingressClassResource.default"
    value = "true"
  }
}

resource "helm_release" "cert_manager" {
  name             = "cert-manager"
  repository       = "https://charts.jetstack.io"
  chart            = "cert-manager"
  namespace        = "cert-manager"
  create_namespace = true

  set {
    name  = "crds.enabled"
    value = "true"
  }
}
```

**MetalLB-Adressbereich, bewusst außerhalb des Hypervisor-eigenen Auto-Zuweisungs-Pools platziert (siehe Diagnosearbeit oben):**

```hcl
resource "kubernetes_manifest" "metallb_pool" {
  manifest = {
    apiVersion = "metallb.io/v1beta1"
    kind       = "IPAddressPool"
    metadata   = { name = "default", namespace = "metallb-system" }
    spec       = { addresses = ["10.0.1.240-10.0.1.250"] }
  }
  depends_on = [helm_release.metallb]
}
```

**Let's-Encrypt-`ClusterIssuer` per DNS-01-Challenge über Cloudflare: die Lösung für ein öffentlich vertrauenswürdiges Zertifikat ohne öffentliche Erreichbarkeit des Dienstes:**

```hcl
resource "kubernetes_manifest" "letsencrypt_issuer" {
  manifest = {
    apiVersion = "cert-manager.io/v1"
    kind       = "ClusterIssuer"
    metadata   = { name = "letsencrypt-cloudflare" }
    spec = {
      acme = {
        server = "https://acme-v02.api.letsencrypt.org/directory"
        email  = "platform@example.com"
        privateKeySecretRef = { name = "letsencrypt-cloudflare-account-key" }
        solvers = [{
          dns01 = {
            cloudflare = {
              apiTokenSecretRef = {
                name = "cloudflare-api-token"
                key  = "api-token"
              }
            }
          }
        }]
      }
    }
  }
  depends_on = [kubernetes_secret.cloudflare_api_token]
}
```

## Ergebnis

Die Anwendung läuft auf Kubernetes, vollständig in Terraform definiert, mit vertrauenswürdigem TLS-Zertifikat, verifizierter Datenpersistenz über Pod-Neustarts hinweg, und ganz ohne manuelle Zertifikats-Vertrauenskonfiguration pro Client. Die aufgebaute Ingress-/TLS-/Load-Balancing-Plattform ist für jede weitere interne App-Migration wiederverwendbar, keine Einzelfall-Lösung nur für dieses eine Projekt.

## Gezeigte Fähigkeiten

Terraform (Multi-Workspace-Design, Provider-Grenzen wie CRD-abhängige Plan-Reihenfolge), Helm-Chart-Erstellung und -Aufräumen, Kubernetes-Netzwerk (Services, Ingress, LoadBalancer/MetalLB, DNS), PKI/TLS (`cert-manager`, ACME DNS-01, Lifecycle einer privaten CA), systemübergreifende Netzwerk-Diagnose (Firewall, Hypervisor-IPAM, DNS), GitOps-Troubleshooting (ArgoCD-Reconciliation), sicheres Credential-Design (Least Privilege, eingeschränkte API-Tokens), und angemessene Sicherheits-Triage.

---

[← Zurück zur Übersicht](README.md)
