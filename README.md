# Dispatch City

Visuelles Food-Delivery-System als kumulative Projektarbeit für das Modul
*Verteilte Systeme, Containerisierung* (TEKO, VSC-01).

Das System wächst über fünf Ausbaustufen von zwei Containern in einem lokalen
k3d-Cluster zu einem verteilten System mit RabbitMQ-Event-Pipeline,
CloudNativePG-Datenhaltung und Prometheus/Grafana-Observability. Das
Nuxt-Dashboard zeigt in jeder Stufe eine animierte 2.5D-Stadt und macht den
technischen Ausbaustand sichtbar.

Repository: <https://github.com/scherssim/vsc-dispatch-city>

## Voraussetzungen

| Werkzeug | Version im Referenzlauf | Zweck |
| --- | --- | --- |
| Docker Desktop | aktuell | Container-Runtime für k3d und Image-Builds |
| k3d | 5.x | lokaler Kubernetes-Cluster |
| kubectl | 1.30+ | Cluster-Zugriff, Kustomize ist eingebaut |
| Helm | 3.x | CloudNativePG-Operator und kube-prometheus-stack |
| Go | 1.25 | Build der sieben Services |
| Node.js / npm | 20+ | Build des Nuxt-Dashboards |
| Git | beliebig | Repository |

Zusätzliche Werkzeuge wie `make` oder `jq` werden nicht benötigt. Alle Befehle
sind für Bash (macOS, Linux, WSL, Git Bash) und für Windows PowerShell
angegeben. Der Referenzlauf entstand auf Windows 11 mit Docker Desktop.

Cluster, Kontext und Namespace sind über alle Blöcke fix:

```text
Cluster:   teko-k8s          (1 Server, 2 Agents)
Kontext:   k3d-teko-k8s
Namespace: food-delivery     (Monitoring liegt in monitoring)
```

## Komponenten

| Komponente | Technologie | Aufgabe |
| --- | --- | --- |
| `apps/dashboard` | Nuxt 4, PixiJS | 2.5D-Stadt, Systemansicht, Live-Events via REST und SSE |
| `cmd/control-api` | Go | REST, SSE, Health, Metrics; im Standalone-Modus die Simulation |
| `cmd/customer-simulator` | Go | erzeugt Kundenidentitäten und Bestellungen als Events |
| `cmd/restaurant-worker` | Go | konkurrierende Kitchen-Consumer je Restaurant |
| `cmd/courier-simulator` | Go | Kurier-Pods, Fahrten und Zustellungen |
| `cmd/order-worker` | Go | einziger Schreiber des fachlichen Zustands, idempotente Projektion |
| `cmd/migrate` | Go | Schema-Migration als Kubernetes-Job |
| `cmd/cluster-observer` | Go | liest Workload-Zustände für die Systemansicht, RBAC auf `food-delivery` begrenzt |
| RabbitMQ | `rabbitmq:4.3.5-management-alpine` | Topic-Exchange `food.events`, Queues, DLQ |
| PostgreSQL | CloudNativePG 1.30.0 | Primary/Standby-Cluster `food-delivery-db` |
| Monitoring | kube-prometheus-stack 88.1.3 | Prometheus, Grafana, ServiceMonitors |

## Ausbaustufen

Jedes Overlay baut auf dem vorherigen auf. Ein Overlay ersetzt den Vorgänger
nicht, es erweitert ihn.

| Overlay | Block | Inhalt |
| --- | --- | --- |
| `deploy/overlays/block-03-standalone` | 3 | Dashboard und Control API als eigene Images, Services, ConfigMap, Probes |
| `deploy/overlays/block-04-ingress` | 4 | Traefik-Ingress, gemeinsamer Einstiegspunkt, zwei Dashboard-Replikas |
| `deploy/overlays/block-05-messaging` | 5 | RabbitMQ, Exchange, Queues, DLQ, vier fachliche Worker-Workloads |
| `deploy/overlays/block-06-persistence` | 6 | CloudNativePG-Cluster, Migration-Job, persistente Order-Projektion |
| `deploy/overlays/block-07-observability` | 7 | cluster-observer, kube-prometheus-stack, ServiceMonitors, RabbitMQ-Metrics-Plugin, Grafana-Dashboard |

Readiness, Rollback und HPA werden bewusst nicht am laufenden Fachsystem
demonstriert, sondern im isolierten NGINX-Lab unter `labs/block-07`
(Namespace `betrieb-lab`). Siehe Abschnitt „Resilienz- und Skalierungs-Lab".

Der Zwischenstand jedes Blocks ist als eigener Commit nachvollziehbar.

## Vollständiger Stand aufbauen

Die folgende Reihenfolge baut den finalen Stand (Block 7) von einem leeren
Rechner aus auf. Die Schritte 3 bis 5 hängen voneinander ab: der Operator muss
die CRD kennen, bevor das Overlay eine `Cluster`-Resource anlegt, und der
Monitoring-Start prüft, ob der Datenbank-Cluster bereits existiert.

### 1. Cluster anlegen

```bash
k3d cluster create teko-k8s --agents 2
kubectl config use-context k3d-teko-k8s
```

Traefik bringt k3s als Ingress-Controller in `kube-system` bereits mit.

### 2. Images bauen und importieren

Das Skript baut die sieben Go-Images. Das Dashboard-Image entsteht aus einem
eigenen Dockerfile und muss separat gebaut werden, bevor der Import läuft –
sonst bricht `load-images` ab, weil ein Image im lokalen Daemon fehlt.

```bash
# Bash
docker build -t food-delivery-dashboard:local ./apps/dashboard
./scripts/build-images.sh
CLUSTER=teko-k8s ./scripts/load-images.sh
```

```powershell
# PowerShell
docker build -t food-delivery-dashboard:local ./apps/dashboard
./scripts/build-images.ps1
./scripts/load-images.ps1 -Cluster teko-k8s
```

Das Image `food-delivery-cluster-observer:local` wird in Schritt 5 automatisch
gebaut und importiert.

### 3. CloudNativePG-Operator installieren

```bash
# Bash
./platform/cloudnative-pg/install.sh
```

```powershell
# PowerShell
./platform/cloudnative-pg/install.ps1
```

Das Skript pinnt Chart 0.29.0 (Operator 1.30.0) und wartet, bis die CRD
`clusters.postgresql.cnpg.io` established ist.

### 4. Anwendung und Datenbank ausrollen

```bash
kubectl --context k3d-teko-k8s apply -k deploy/overlays/block-06-persistence
kubectl --context k3d-teko-k8s -n food-delivery wait --for=condition=Ready cluster/food-delivery-db --timeout=5m
kubectl --context k3d-teko-k8s -n food-delivery wait --for=condition=Available deployment --all --timeout=5m
```

### 5. Observability ergänzen

```bash
# Bash
sh platform/monitoring/start-course.sh
```

```powershell
# PowerShell
./platform/monitoring/start-course.ps1
```

Der Start baut und importiert das Observer-Image, installiert den
kube-prometheus-stack per Helm, wendet das Block-7-Overlay an und wartet auf
cluster-observer, RabbitMQ und Grafana.

### 6. Zugänge öffnen

Jeder Port-Forward belegt ein eigenes Terminal.

```bash
kubectl --context k3d-teko-k8s -n kube-system   port-forward service/traefik            8080:80
kubectl --context k3d-teko-k8s -n food-delivery port-forward service/rabbitmq           15672:15672
kubectl --context k3d-teko-k8s -n monitoring    port-forward service/monitoring-grafana 3000:80
```

| Oberfläche | Adresse | Zugang |
| --- | --- | --- |
| Dispatch City | <http://localhost:8080/> | – |
| Control API | <http://localhost:8080/api/v1/snapshot> | – |
| RabbitMQ Management | <http://localhost:15672/> | `delivery` / `delivery` |
| Grafana | <http://localhost:3000/> | `admin` / `delivery`, Dashboard "Dispatch City – Betrieb" |

Die Zugangsdaten sind bewusst fixe Kurswerte und nicht für einen Betrieb
ausserhalb dieses lokalen Labs gedacht.

## Einzelne Ausbaustufe deployen

Zum Vorführen einer früheren Stufe genügt das jeweilige Overlay. Die Stufen 3
bis 5 brauchen weder Operator noch Monitoring.

```bash
kubectl --context k3d-teko-k8s apply -k deploy/overlays/block-04-ingress
kubectl --context k3d-teko-k8s -n food-delivery rollout status deployment/dashboard --timeout=180s
```

## Smoke-Test

Die Prüfung setzt die drei Port-Forwards aus Schritt 6 voraus und deckt alle
fünf Ausbaustufen ab.

### Workloads bereit

```bash
kubectl --context k3d-teko-k8s -n food-delivery get deploy,statefulset,job,pvc
kubectl --context k3d-teko-k8s -n food-delivery get cluster food-delivery-db
kubectl --context k3d-teko-k8s -n food-delivery get pods -L cnpg.io/instanceRole
```

Erwartet: alle Workloads bereit, `food-delivery-db` mit `INSTANCES 2 / READY 2`,
genau ein Pod mit Rolle `primary`.

### Einstiegspunkt und Routen

```bash
# Bash
for url in / /api/v1/snapshot /health/ready /metrics; do
  curl -s -o /dev/null -w "%{http_code} $url\n" "http://localhost:8080$url"
done
```

```powershell
# PowerShell
foreach ($u in "/", "/api/v1/snapshot", "/health/ready", "/metrics") {
  "{0} {1}" -f (Invoke-WebRequest -UseBasicParsing "http://localhost:8080$u").StatusCode, $u
}
```

Erwartet: viermal HTTP 200.

### Eventfluss und Persistenz

```bash
# Bestellung erzeugen und danach in PostgreSQL wiederfinden
curl -s -X POST http://localhost:8080/api/v1/orders

kubectl --context k3d-teko-k8s -n food-delivery exec rabbitmq-0 -- \
  rabbitmqctl list_queues name consumers messages_ready

PRIMARY=$(kubectl --context k3d-teko-k8s -n food-delivery get pod \
  -l cnpg.io/cluster=food-delivery-db,cnpg.io/instanceRole=primary \
  -o jsonpath='{.items[0].metadata.name}')
kubectl --context k3d-teko-k8s -n food-delivery exec "$PRIMARY" -- \
  psql -d delivery -c 'SELECT id,status,updated_at FROM orders ORDER BY updated_at DESC LIMIT 5;'
```

Erwartet: der Topic-Exchange `food.events` mit neun Queues und aktiven
Consumern, die neue Bestellung im Dashboard-Event-Stream und als Zeile in
`orders`.

### Idempotenz

```bash
# Bash
./scripts/lab-idempotency.sh
```

```powershell
# PowerShell
./scripts/lab-idempotency.ps1
```

Publiziert dieselbe Nachricht zweimal. Erwartet: genau eine Zeile in `orders`
und ein Eintrag in `processed_events` – die zweite Zustellung wird verworfen.

### Observability

```bash
kubectl --context k3d-teko-k8s -n monitoring get pods
kubectl --context k3d-teko-k8s -n food-delivery get servicemonitor
kubectl --context k3d-teko-k8s -n food-delivery rollout status deployment/cluster-observer --timeout=120s
```

Erwartet: Prometheus und Grafana bereit, ServiceMonitors für Anwendung und
RabbitMQ vorhanden, Systemansicht im Dashboard zeigt Workloads mit Readiness.

## Resilienz- und Skalierungs-Lab

Readiness-Verhalten bei Rolling Updates, Rollback und horizontale Skalierung
laufen in einem eigenen Namespace, damit die Demo den fachlichen Zustand nicht
verfälscht.

```bash
kubectl --context k3d-teko-k8s apply -f labs/block-07/web.yaml
kubectl --context k3d-teko-k8s apply -f labs/block-07/hpa.yaml
kubectl --context k3d-teko-k8s -n betrieb-lab get hpa -w
```

In einem zweiten Terminal 150 Sekunden künstliche CPU-Last erzeugen:

```bash
# Bash
sh labs/block-07/load.sh
```

```powershell
# PowerShell
./labs/block-07/load.ps1
```

Erwartet: Der HPA meldet eine gemessene CPU-Auslastung statt `<unknown>` und
skaliert `lab-web` von 2 auf bis zu 4 Replikas. Nach dem Ende der Last fährt er
nach dem Stabilisierungsfenster von 60 Sekunden wieder herunter.

Aufräumen:

```bash
kubectl --context k3d-teko-k8s delete namespace betrieb-lab
```

## Reset

```bash
# Anwendung zurücksetzen, Cluster und Operator bleiben bestehen
kubectl --context k3d-teko-k8s delete namespace food-delivery

# Monitoring zusätzlich entfernen
helm --kube-context k3d-teko-k8s -n monitoring uninstall monitoring

# vollständig, inklusive Operator und Cluster
helm --kube-context k3d-teko-k8s -n cnpg-system uninstall cnpg
k3d cluster delete teko-k8s
```

Mit dem Namespace verschwinden auch die PVCs von RabbitMQ und PostgreSQL. Der
fachliche Zustand ist danach leer; der Wiederaufbau beginnt bei Schritt 4.

## Bewusste Abweichungen vom Kursstand

**RabbitMQ-Probes und CPU-Limit.** Der Kursstand setzt für die Liveness Probe
`timeoutSeconds: 5`. Auf dem Referenzrechner braucht `rabbitmq-diagnostics`
gemessene 15 bis 19 Sekunden, der Broker startet in rund 92 Sekunden. Die Probe
tötete den Pod deshalb in einer Endlosschleife und `control-api` blieb im
CrashLoop. `deploy/overlays/block-06-persistence/rabbitmq-probe-patch.yaml`
setzt daher `timeoutSeconds` auf 15 beziehungsweise 25, erhöht
`failureThreshold` auf 6 und das CPU-Limit auf 2. Die Werte sind im Repository
korrigiert, nicht nur zur Laufzeit gepatcht.

## Dokumentation

- `docs/architecture.md` – Komponenten, Eventfluss, Datenhaltung und die
  wichtigsten Entscheidungen je Ausbaustufe
- `docs/komponenten.md` – Komponentendiagramm zu `docs/architecture.md`,
  separat, damit es in voller Breite lesbar ist
- `docs/block-03-abnahme.md` bis `docs/block-07-abnahme.md` –
  Abnahmeprotokolle mit gemessenen Werten, eines pro Ausbaustufe
- `labs/block-07` – NGINX-Lab für Readiness, Rollback und HPA

## Hilfsmittel und KI-Unterstützung

Offengelegt gemäss Ehrenkodex der Leistungsbeurteilung VSC-01.

**Vorgegebenes Material.** Die Bausteine der Blöcke 3 bis 7 stammen aus den
Kursrepositories von Patrick Michel und wurden per Installationsskript in
diesen Projektstand übernommen: `vsc-dispatch-city-03-foundation` v1.0.0,
`-04-ingress` v1.1.1, `-05-messaging` v1.1.0, `-06-persistence` v1.1.0 und
`-07-observability` v1.1.0. Die Git-Historie dieses Repositories zeigt pro
Block, was übernommen und was danach selbst geändert wurde.

**Standardsoftware.** RabbitMQ als offizielles Image, CloudNativePG und
kube-prometheus-stack als Helm-Charts in den oben gepinnten Versionen.

**KI-Unterstützung.** Claude (Anthropic) wurde eingesetzt für:

- Fehlersuche bei Betriebsproblemen der lokalen Umgebung, insbesondere beim
  RabbitMQ-CrashLoop, bei einer k3d-Node-IP-Kollision nach einem
  Docker-Neustart und bei einer kubeconfig, die auf eine nicht mehr erreichbare
  LAN-Adresse zeigte
- Formulierungshilfe und Struktur für dieses README und die Abnahmeprotokolle:
  - [`docs/block-03-abnahme.md`](docs/block-03-abnahme.md)
  - [`docs/block-04-abnahme.md`](docs/block-04-abnahme.md)
  - [`docs/block-05-abnahme.md`](docs/block-05-abnahme.md)
  - [`docs/block-06-abnahme.md`](docs/block-06-abnahme.md)
  - [`docs/block-07-abnahme.md`](docs/block-07-abnahme.md)
- Erklärung von Kubernetes- und CloudNativePG-Verhalten als Ergänzung zur
  offiziellen Dokumentation

Alle Befehle, Messwerte und Beobachtungen in den Abnahmeprotokollen stammen aus
eigenen Läufen auf dem eigenen Cluster. Übernommener Code wurde vor dem Commit
gelesen und im Cluster verifiziert.
