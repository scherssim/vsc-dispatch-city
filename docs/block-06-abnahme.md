# Block 6 – Abnahme

Durchgeführt am 31.08.2026 auf demselben k3d-Cluster `teko-k8s` (Kontext `k3d-teko-k8s`),
drei Nodes Ready. Der Namespace `food-delivery` aus Block 5 lief noch und wurde in place
erweitert — kein Neuaufbau.

Ausgangsstand: eigener Projektstand nach Block 5 (Commit `AB5: RabbitMQ Event Pipeline –
Aufgaben 4 und 5`). Darüber installiert: `SwitzerChees/vsc-dispatch-city-06-persistence`,
Release `v1.1.0`, geklont nach `../vsc-dispatch-city-06-persistence`. Alle Dateien sind
byte-identisch zum Kurspaket übernommen (Commit `AB6: Helm, Operator und CloudNativePG –
Integration`).

Alle sechs Aufgaben des Arbeitsblatts sind durchgeführt. Zeiten in UTC, sofern nicht anders
angegeben.

## Was der Baustein mitbringt

| Kategorie | Pfade |
| --------- | ----- |
| Operator | `platform/cloudnative-pg/` – Helm-Installation nach `cnpg-system`, `values-course.yaml` |
| Overlay | `deploy/overlays/block-06-persistence/` – `postgres.yaml` (Cluster-Resource), `database-patch.yaml`, `migrate-job.yaml`, `rabbitmq-probe-patch.yaml` |
| Code | `cmd/migrate/`, `internal/persistence/` mit `migrations/001_initial.sql` und `repository.go`; `cmd/order-worker` und `cmd/control-api` erweitert |
| Lab | `scripts/lab-idempotency.{sh,ps1}` – publiziert dieselbe `event_id` zweimal |

## Systemgrenze nach Block 6

| Komponente | Rolle | Instanzen |
| ---------- | ----- | --------- |
| `cnpg` (Helm-Release in `cnpg-system`) | Operator 1.30.0, Chart 0.29.0 | 1 |
| `food-delivery-db` | CloudNativePG-Cluster, PostgreSQL 18.4, je 1-GiB-PVC (`local-path`) | 2 (Primary + Standby) |
| `food-delivery-db-rw` / `-ro` / `-r` | vom Operator erzeugte Services | – |
| `migrate` | Job, legt das Schema an (`ttlSecondsAfterFinished: 600`) | einmalig |
| `order-worker` | einziger Schreiber des fachlichen Zustands | wie Block 5 |
| `control-api` | liest den Snapshot aus PostgreSQL | 2 |

Die Persistenzlücke aus Block 5 ist damit geschlossen: `control-api` darf jetzt mehrere
Replicas haben, weil der Zustand nicht mehr im Prozess liegt.

## Aufgabe 1 – Baustein integrieren und Wunschzustand lesen

```powershell
& '..\vsc-dispatch-city-06-persistence\install.ps1' -Target '.'
kubectl kustomize deploy/overlays/block-06-persistence | Out-File block6.yaml
```

Das Overlay rendert fehlerfrei: 870 Zeilen, 26 Objekte. Es enthält genau eine
`Cluster`-Resource, aber weder StatefulSet noch Deployment für die Datenbank — nur die drei
StatefulSets und sechs Deployments der Anwendung aus Block 5.

| Prognose vor dem Deployment | Begründung aus dem YAML |
| --------------------------- | ----------------------- |
| zwei DB-Instanzen auf getrennten Nodes | `Cluster/food-delivery-db`, `spec.instances: 2`; `spec.affinity.podAntiAffinityType: required`, `topologyKey: kubernetes.io/hostname` |
| Image `ghcr.io/cloudnative-pg/postgresql:18.4-system-trixie` | `spec.imageName`; nicht zu verwechseln mit dem Operator-Image `cloudnative-pg:1.30.0` |
| Writes über `food-delivery-db-rw` | Secret `database-url`: `postgres://delivery:…@food-delivery-db-rw:5432/delivery`. Der Service steht nirgends im Overlay, der Operator legt ihn an |
| DB-Pods erzeugt der Operator direkt | nach dem Apply bestätigt: `ownerReferences[0].kind = Cluster`, `apiVersion postgresql.cnpg.io/v1` |

## Aufgabe 2 – Chart prüfen und Operator installieren

```bash
helm show chart  cnpg/cloudnative-pg --version 0.29.0
helm show values cnpg/cloudnative-pg --version 0.29.0
./platform/cloudnative-pg/install.ps1
helm -n cnpg-system list
kubectl get crd clusters.postgresql.cnpg.io
```

| Wert | Default oder Override? | Warum |
| ---- | ---------------------- | ----- |
| Chart 0.29.0 / appVersion 1.30.0 | Kurs-Pin | ohne `--version` zieht Helm immer die neueste Version; der Pin macht den Stand reproduzierbar |
| Operator-Ressourcen 50m/96Mi bis 300m/256Mi | echter Override (Default `resources: {}`) | planbare Reservierung, kann die knappe Host-CPU beim Failover nicht monopolisieren |
| `monitoring.podMonitorEnabled: false` | Chart-Default, bewusst wiederholt | ein PodMonitor bräuchte die CRDs des Prometheus-Operators, die in Block 6 noch fehlen |

Helm installiert einmalig die Steuerungsebene: Deployment, RBAC, Webhook-Service und elf
CRDs. Der Operator betreibt danach laufend die Datenbank und führt Pods, PVCs, Services und
TLS-Secrets gegen die `Cluster`-Resource nach. Helm ist also der Installer mit einem
einmaligen Zustand, der Operator ein dauernder Regelkreis — `helm upgrade` ersetzt keinen
Primary.

## Aufgabe 3 – Datenbank deployen und Service-Routing

```bash
./scripts/build-images.ps1
./scripts/load-images.ps1 -Cluster teko-k8s
kubectl --context k3d-teko-k8s apply -k deploy/overlays/block-06-persistence
kubectl --context k3d-teko-k8s -n food-delivery wait --for=condition=Ready cluster/food-delivery-db --timeout=5m
kubectl --context k3d-teko-k8s -n food-delivery get pods -L cnpg.io/instanceRole
kubectl --context k3d-teko-k8s -n food-delivery get endpointslice -l kubernetes.io/service-name=food-delivery-db-rw -o wide
```

`get cluster food-delivery-db` meldet `INSTANCES 2 / READY 2`. `food-delivery-db-1` läuft auf
`server-0` (10.42.0.43), `food-delivery-db-2` auf `agent-0` (10.42.1.43).

| Service | Selector | Endpoint zum Messzeitpunkt | Zweck |
| ------- | -------- | -------------------------- | ----- |
| `food-delivery-db-rw` | `cnpg.io/instanceRole=primary` | 10.42.0.43 (db-1) | alle Schreibzugriffe, `DATABASE_URL` von `order-worker` und `control-api` |
| `food-delivery-db-ro` | `cnpg.io/instanceRole=replica` | 10.42.1.43 (db-2) | Nur-Lese-Last; ein INSERT scheitert mit `cannot execute INSERT in a read-only transaction` |
| `food-delivery-db-r` | `cnpg.io/podRole=instance` | beide | Lesen ohne Rollenpräferenz, ungeeignet für den aktuellsten Stand |

Der Order Worker darf keinen Pod-Namen als Host verwenden: Der Name kennt die Rolle nicht.
`food-delivery-db-1` war beim Deployment Primary, nach dem Failover in Aufgabe 6 Replica.
Ausserdem ändert sich die Pod-IP bei jedem Neustart (10.42.0.43 → 10.42.0.44). Stabil ist nur
der Service `-rw`, dessen Label der Operator umhängt.

### Stolperfalle: RabbitMQ-Probe aus dem Kurspaket

`rabbitmq-probe-patch.yaml` setzt `timeoutSeconds: 5`. `rabbitmq-diagnostics` braucht auf
diesem Rechner gemessene 15 bis 19 s, der Broker startet in rund 92 s. Die Liveness Probe
tötete den Pod in einer Endlosschleife, `control-api` blieb im CrashLoop. In Block 6 wurde
das zur Laufzeit mit `kubectl patch statefulset rabbitmq` behoben (Timeouts 25/15 s,
`limits.cpu: 2`); ein erneutes `apply -k` stellte den Kurswert wieder her. Seit dem Commit
zu Block 7 steht der korrigierte Wert im Repository.

## Aufgabe 4 – Persistenz nachweisen

Bestellung über `POST /api/v1/orders` erzeugt:

```
id          afebed53-5f21-4aa1-ae82-72beac4c2866
Kunde       customer-agent-10001, Restaurant restaurant-bowl, angelegt 18:10:07Z
Status      delivered, progress 1, updated_at 2026-08-31 18:10:30Z
```

Derselbe Stand im Dashboard-Snapshot (`/api/v1/snapshot`) und in PostgreSQL. Danach nur die
zustandsführenden Anwendungen neu gestartet:

```bash
kubectl --context k3d-teko-k8s -n food-delivery rollout restart deployment/order-worker deployment/control-api
kubectl --context k3d-teko-k8s -n food-delivery rollout status deployment/order-worker
kubectl --context k3d-teko-k8s -n food-delivery rollout status deployment/control-api
```

Beide melden `successfully rolled out`, neue Pods `control-api-846987755f-*` und
`order-worker-5dc9d56755-*` mit `RESTARTS 0`. Die Bestellung war sofort wieder im Snapshot und
in der Datenbank.

- **Persistenz** liefert PostgreSQL: `Repository.Project` schreibt jedes Event in einer
  Transaktion nach `processed_events`, `order_events` und `orders`. Die Daten liegen auf den
  PVCs der DB-Pods, ausserhalb der Anwendungspods.
- **Rekonstruktion der Anzeige** leistet `control-api`: Beim Start `LoadSnapshot()` und
  `engine.Hydrate()`, danach hält `refreshFromDatabase()` den Stand nach. Das Dashboard ist
  zustandslos.

Beobachtung: `control-api` zeigt nur die 18 zuletzt aktualisierten Bestellungen
(`maxVisibleOrders = 18`). Eine halbe Stunde später war die Bestellung aus dem Sichtfenster
gerutscht, in PostgreSQL aber unverändert vorhanden — kein Datenverlust.

## Aufgabe 5 – Doppelte Nachricht, einfache Wirkung

```bash
kubectl --context k3d-teko-k8s -n food-delivery port-forward service/rabbitmq 15672:15672
./scripts/lab-idempotency.ps1
```

Das Lab publizierte `EVENT_ID 7e17f14f-456e-43ab-8ddc-e97676e6780e` mit
`ORDER_ID lab-idempotency-20260831T181331` zweimal; RabbitMQ bestätigte beide Male
`routed=true`.

| Messpunkt | Prognose | Gemessen |
| --------- | -------- | -------- |
| `processed_events` für die `event_id` | 1 | 1 |
| `orders` für die `order_id` | 1 | 1 |
| `order_events` für die `event_id` | 1 | 1 |

Log des `order-worker`: 18:13:33 `event projected to PostgreSQL`, 18:13:36
`duplicate event ignored by PostgreSQL`.

Der Mechanismus: `processed_events.event_id` ist Primary Key.
`INSERT … ON CONFLICT (event_id) DO NOTHING RETURNING event_id` liefert bei der zweiten
Zustellung keine Zeile, die Projektion läuft nicht. Claim und Projektion stehen in derselben
Transaktion — sonst gäbe es zwei Fehlerfenster: Markierung ohne Wirkung (Event verloren) oder
Wirkung ohne Markierung (Event doppelt). Erst nach `tx.Commit()` wird die RabbitMQ-Nachricht
bestätigt. Aus „mindestens einmal zugestellt" wird so „genau einmal wirksam".

## Aufgabe 6 – Failover beobachten

Vorher: Primary `food-delivery-db-1` (10.42.0.43, `server-0`), Standby `food-delivery-db-2`
(10.42.1.43, `agent-0`). EndpointSlice `food-delivery-db-rw-slp9b` zeigte ausschliesslich auf
10.42.0.43. Cluster-Status „Cluster in healthy state", 2/2 ready.

```bash
# Terminal 1 und 2: Beobachtung
kubectl --context k3d-teko-k8s -n food-delivery get pods -L cnpg.io/instanceRole -w
kubectl --context k3d-teko-k8s -n food-delivery get endpointslice -l kubernetes.io/service-name=food-delivery-db-rw -w

# Terminal 3: harter Ausfall (nur im lokalen Kurscluster)
kubectl --context k3d-teko-k8s -n food-delivery delete pod food-delivery-db-1 --grace-period=0 --force
```

Gemessen mit 0,3-s-Polling ab dem Force-Delete (18:14:56Z):

| Ereignis | nach |
| -------- | ---- |
| anderer Pod trägt `cnpg.io/instanceRole=primary` | 32,1 s |
| `-rw`-Endpoint auf 10.42.1.43 umgehängt | 33,5 s |
| neu erzeugter `food-delivery-db-1` (10.42.0.44) als Replica gelabelt | 21 s nach seiner Erstellung |
| Cluster wieder „healthy", 2/2 ready | rund 2 min |

Neuer Primary ist `food-delivery-db-2` auf `agent-0`, also ein echter Node-Wechsel.
`currentPrimary` seit 18:15:25Z. An `DATABASE_URL`, Deployments oder Pods der Anwendung wurde
nichts geändert.

Die Bestellung aus Aufgabe 4 war unverändert vorhanden (`delivered / 1 / 18:10:30`), jetzt vom
neuen Primary beantwortet. Die Idempotenz-Datensätze aus Aufgabe 5 ebenfalls (1/1), insgesamt
37 Bestellungen in `orders`. Dashboard und Simulation liefen durch: `http://localhost:8080`
antwortete mit 200, `running = true`.

## Was Block 6 nicht löst

**Bewiesen** ist Verfügbarkeit gegen den Ausfall einer Instanz bzw. eines Nodes: rund 34 s
Schreibpause, danach ohne manuellen Eingriff weiter. Ebenso Dauerhaftigkeit über
Pod-Neustarts (Aufgabe 4) und Effectively-once-Verarbeitung (Aufgabe 5).

**Nicht bewiesen** ist Datensicherheit. Es gibt kein Backup: keinen `spec.backup`-Abschnitt,
keine `ScheduledBackup`-Resource, kein WAL-Archiv, kein Object-Storage-Ziel. Beide Kopien
liegen auf `local-path`-PVCs desselben Docker-Hosts. Gegen versehentliches `DELETE` oder eine
fehlerhafte Migration hilft Replikation nicht — sie repliziert den Fehler mit.

**Offen** ist das RPO im Failover selbst. Die Replikation ist asynchron (weder
`minSyncReplicas` noch `maxSyncReplicas` gesetzt); beim harten Ausfall können die letzten
nicht übertragenen Transaktionen verlorengehen. Der Test hat das nicht widerlegt, weil die
geprüfte Bestellung Minuten vor dem Ausfall committet war. Dafür bräuchte es Schreiblast im
Moment des Ausfalls, Backup mit Point-in-Time-Recovery und einen Restore-Test.

## Abnahmestand

```
Overlay          deploy/overlays/block-06-persistence   26 Objekte, 870 Zeilen gerendert
Operator         Helm-Release cnpg, Chart 0.29.0, Operator 1.30.0, Namespace cnpg-system
Datenbank        food-delivery-db, PostgreSQL 18.4, 2/2 ready, Anti-Affinity erfüllt
Schema           orders, order_events, processed_events, customers, couriers, restaurants
Persistenz       Bestellung afebed53-… übersteht Rollout von order-worker und control-api
Idempotenz       doppelt publiziert -> processed_events 1, orders 1, order_events 1
Failover         Rolle nach 32,1 s, -rw nach 33,5 s, db-1 -> db-2, Node server-0 -> agent-0
Endstand         232 Bestellungen, 7156 processed_events, control-api 2 Replicas
Abweichungen     RabbitMQ-Probes zur Laufzeit gepatcht (im Repo erst mit Block 7 korrigiert)
```

## Aufräumen

```powershell
# Port-Forwards beenden (Strg+C im jeweiligen Terminal)
# Der Cluster teko-k8s und der Namespace food-delivery bleiben für Block 7 bestehen.
# Vollständig zurücksetzen:
kubectl --context k3d-teko-k8s delete namespace food-delivery
helm --kube-context k3d-teko-k8s -n cnpg-system uninstall cnpg
```
