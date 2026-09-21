# Block 7 – Abnahme

Durchgeführt am 08.09.2026 auf demselben k3d-Cluster `teko-k8s` (Kontext `k3d-teko-k8s`),
drei Nodes Ready. Der Namespace `food-delivery` aus Block 6 lief noch, `food-delivery-db`
healthy mit zwei Instanzen.

Ausgangsstand: eigener Projektstand nach Block 6 (Commit `AB6: Helm, Operator und
CloudNativePG – Integration`). Darüber installiert:
`SwitzerChees/vsc-dispatch-city-07-observability`, Release `v1.1.0`, geklont nach
`../vsc-dispatch-city-07-observability`.

Vor dem Start zwei Umgebungsprobleme behoben: Nach dem Neustart von Docker Desktop hatte
`k3d-teko-k8s-server-0` die falsche Container-IP (`k3d cluster stop` und `start` nacheinander
räumt das auf), und `~/.kube/config` zeigte auf `host.docker.internal:53512`, das auf eine
nicht mehr erreichbare LAN-Adresse auflöste — umgestellt auf `127.0.0.1:53512`.

Alle fünf Aufgaben des Arbeitsblatts sind durchgeführt. Zeiten in UTC.

## Was der Baustein mitbringt

| Kategorie | Pfade |
| --------- | ----- |
| Monitoring | `platform/monitoring/` – kube-prometheus-stack 88.1.3 als Helm-Release `monitoring`, `values-light.yaml`, `start-course.{sh,ps1}` |
| Overlay | `deploy/overlays/block-07-observability/` – cluster-observer, ServiceMonitors für Anwendung und RabbitMQ, PodMonitor für PostgreSQL, Grafana-Dashboard, RabbitMQ-Prometheus-Plugin |
| Code | `cmd/cluster-observer/`, `internal/cluster/observer.go` – lesender Observer, RBAC auf `food-delivery` begrenzt |
| Lab | `labs/block-07/` – NGINX-Deployment `lab-web` im Namespace `betrieb-lab`, HPA, Lastskript |

## Systemgrenze nach Block 7

Das Fachsystem ist gegenüber Block 6 unverändert, es wird nur beobachtet. Readiness, Rollback
und HPA werden bewusst im isolierten Namespace `betrieb-lab` geübt, damit die Demo den
fachlichen Zustand nicht verfälscht.

| Komponente | Namespace | Aufgabe |
| ---------- | --------- | ------- |
| Prometheus, Grafana, kube-state-metrics | `monitoring` | Metriken sammeln, speichern, anzeigen |
| `cluster-observer` | `food-delivery` | liefert Workload-Zustände an die Systemansicht im Dashboard |
| ServiceMonitors / PodMonitor | `food-delivery` | sagen Prometheus, welche `/metrics`-Endpunkte es abfragen soll |
| `lab-web` + HPA | `betrieb-lab` | Übungsobjekt für Probes, Rollout und Skalierung |

## Aufgabe 1 – Das Monitoring öffnen

```powershell
& '..\vsc-dispatch-city-07-observability\install.ps1' -Target '.'
./platform/monitoring/start-course.ps1
kubectl -n monitoring port-forward service/monitoring-grafana 3000:80
kubectl -n kube-system port-forward service/traefik 8081:80
```

Grafana-Dashboard „Dispatch City – Betrieb", Datasource-UID `prometheus` (gegengeprüft über
den Datasource-Proxy von Grafana).

| Anzeige | Query | Art der Zahl | Gemessen |
| ------- | ----- | ------------ | -------- |
| Wartende Nachrichten je Restaurant | `rabbitmq_queue_messages_ready{namespace="food-delivery",queue=~"restaurant.*"}` | Bestand | alle drei Queues 0 (07:46:44Z) |
| Verarbeitete Events pro Sekunde | `sum(rate(food_delivery_events_consumed_total{namespace="food-delivery"}[1m]))` | Durchsatz | 2.51, 2.48, 2.58, 2.52 (07:44:10Z–07:45:14Z) |
| Offene Bestellungen | `max(food_delivery_active_orders)` | Bestand | konstant 5 |
| Gelieferte Bestellungen | `max(food_delivery_delivered_orders_total)` | monotoner Zähler | 39 → 40 |
| Pizza-Worker bereit | kube-state-metrics | Sollzustand | 1 |

## Aufgabe 2 – Den Rückstau erkennen

Vermutung vorher: RabbitMQ nimmt Bestellungen weiter an, ohne Consumer holt sie niemand ab.
Die Pizza-Queue muss steigen, bowl und curry bleiben bei 0, nichts geht verloren.

```bash
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=0
```

| Zeit | Pizza-Worker bereit | `restaurant.restaurant-pizza` ready |
| ---- | ------------------- | ----------------------------------- |
| 07:46:44Z | 1 → Scale auf 0 | 0 |
| 07:47:11Z | 0 | 2 |
| bis 07:48:55Z | 0 | 3, 5, 8, 10, 13, 14 |
| beim Zurückskalieren | 0 | 16 |

bowl und curry blieben bei identischer Last durchgehend auf 0 — der Rückstau lässt sich genau
der Queue zuordnen, deren Consumer fehlt.

```bash
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=1
kubectl -n food-delivery rollout status deployment/restaurant-pizza
kubectl -n food-delivery logs deployment/restaurant-pizza --tail=10
```

Pod ready um 07:49:31Z, beim nächsten Scrape (07:49:47Z) war die Queue von 16 auf 0. Das Log
zeigt zwischen 07:49:34Z und 07:49:38Z sieben `order processed`, rund eine Bestellung pro
0,9 s.

Richtige erste Massnahme ist den fehlenden Consumer zurückzubringen, nicht an RabbitMQ zu
schrauben oder die Bestellrate zu drosseln. Es war kein Durchsatzproblem: Die beiden anderen
Queues standen bei gleicher Last auf 0. `replicas=0` ist ein absichtlich geänderter
Sollzustand, Kubernetes hätte den Worker von sich aus nie zurückgeholt.

## Aufgabe 3 – Readiness sichtbar testen

```bash
kubectl apply -f labs/block-07/web.yaml
kubectl -n betrieb-lab rollout status deployment/lab-web
kubectl -n betrieb-lab exec lab-web-678875df8d-cj8tp -- mv /tmp/ready /tmp/not-ready
kubectl -n betrieb-lab get pods
kubectl -n betrieb-lab get endpointslice -o yaml
```

| Zeit | Ereignis |
| ---- | -------- |
| 07:52:18Z | `mv /tmp/ready /tmp/not-ready` |
| 07:52:19Z | Condition `Ready=False` (`periodSeconds 3`, `failureThreshold 1`), `get pods` zeigt 0/1 |
| 07:52:45Z | Datei zurückverschoben |
| 07:52:52Z | wieder 1/1 |

`RESTARTS` und `containerStatuses[0].restartCount` blieben auf 0, es war durchgehend derselbe
Pod. Die Readiness-Probe (`test -f /tmp/ready`) schlug fehl, die Liveness-Probe (`httpGet /`)
nicht. In der EndpointSlice `lab-web-vq6cz` stand `cj8tp` auf `conditions.ready: false`,
`llxwj` unverändert auf `true`. Die Adresse 10.42.0.138 bleibt in der Liste, kube-proxy
verteilt aber nur auf `ready: true`.

Fazit: Liveness startet neu, Readiness nimmt nur aus dem Verkehr.

## Aufgabe 4 – Ein Update stoppen und zurückrollen

```bash
kubectl -n betrieb-lab port-forward service/lab-web 8088:80
kubectl -n betrieb-lab set image deployment/lab-web web=nginx:absichtlich-falsch
kubectl -n betrieb-lab describe pod lab-web-85894cd8fc-dhvwh
```

Events des neuen Pods:

```
Warning  Failed  Failed to pull image "nginx:absichtlich-falsch": rpc error: code = NotFound
                 desc = failed to pull and unpack image "docker.io/library/nginx:absichtlich-falsch":
                 failed to resolve reference "docker.io/library/nginx:absichtlich-falsch": not found
Error: ErrImagePull  ->  BackOff  ->  Error: ImagePullBackOff
```

Der Container startet nie, weil der Tag in der Registry nicht existiert. Deployment-Status
während des Fehlers: `2 desired | 1 updated | 3 total | 2 available | 1 unavailable`,
`Available=True` und `Progressing=True`. Wegen `maxUnavailable: 0` darf Kubernetes einen
alten Pod erst entfernen, wenn ein neuer ready ist; `maxSurge: 1` erlaubt genau einen
zusätzlichen. Die Testseite auf `http://localhost:8088` antwortete durchgehend mit HTTP 200.

```bash
kubectl -n betrieb-lab rollout undo deployment/lab-web
kubectl -n betrieb-lab rollout status deployment/lab-web
```

`rollout undo` um 07:54:18Z setzte `nginx:1.28.0-alpine` zurück und war in rund einer Sekunde
fertig — die alten Pods liefen ja bereits.

## Aufgabe 5 – Ressourcen und automatische Skalierung

`labs/block-07/web.yaml`: `requests.cpu: 100m`, `limits.cpu: 200m`. Der Request ist die
Reservierung, nach der der Scheduler den Node wählt und an der der HPA misst (50 % Ziel =
50 % von 100m). Das Limit ist die harte Obergrenze, an der gedrosselt wird.

Im Leerlauf (`kubectl -n betrieb-lab top pods`): 9m und 10m, HPA meldet 8%/50%.

Eigene Änderung: `maxReplicas` in `labs/block-07/hpa.yaml` von 3 auf 4 erhöht.

```bash
kubectl apply -f labs/block-07/hpa.yaml
kubectl -n betrieb-lab get hpa -w
./labs/block-07/load.ps1        # 150 s CPU-Last
```

| Zeit | CPU / Ziel | Replicas |
| ---- | ---------- | -------- |
| 07:54:47Z | 8% / 50% | 2 (Laststart) |
| 07:55:36Z | 105% / 50% | 3 |
| 07:55:57Z | 108% / 50% | 4 (Obergrenze) |
| danach | 74%, dann 57–60% | 4 |
| 07:57:17Z | 10% (Lastende) | 4 |
| 07:58:44Z | – | 2 |

Events: zweimal `cpu resource utilization (percentage of request) above target`, einmal
`All metrics below target` nach dem `stabilizationWindowSeconds` von 60.

Eine Queue kann trotz niedriger CPU wachsen: In Aufgabe 2 wuchs der Rückstau auf 16, während
`restaurant-pizza` null Replicas und damit null CPU hatte. Auch ein Consumer, der auf Datenbank
oder Netz wartet, verbraucht kaum CPU. Ein CPU-basierter HPA reagiert darauf nicht; dafür
bräuchte es `rabbitmq_queue_messages_ready` als Metrik über einen Prometheus-Adapter oder KEDA.

## RabbitMQ-Probes im Repository korrigiert

Der in Block 6 offen gebliebene Punkt ist mit diesem Block geschlossen:
`deploy/overlays/block-06-persistence/rabbitmq-probe-patch.yaml` trägt jetzt die auf diesem
Rechner funktionierenden Werte (Liveness-Timeout 25 s, Readiness 15 s, `limits.cpu: 2`).
Vorher musste nach jedem `apply -k` von Hand nachgepatcht werden.

## Was Block 7 nicht löst

- Der HPA skaliert nur nach CPU. Für die fachlichen Worker wäre die Queue-Länge die passende
  Metrik; die ist in Grafana sichtbar, steuert aber nichts.
- Es gibt keine Alerting-Regeln. Der Rückstau aus Aufgabe 2 fiel nur auf, weil jemand auf das
  Dashboard schaute.
- Logs werden nicht zentral gesammelt (kein Loki o. Ä.); sie sind nur per `kubectl logs` aus
  laufenden Pods lesbar.
- Monitoring und Anwendung laufen auf denselben drei k3d-Nodes eines Hosts. Fällt der Host,
  fällt die Beobachtung mit.

## Abnahmestand

```
Overlay          deploy/overlays/block-07-observability
Monitoring       Helm-Release monitoring, kube-prometheus-stack 88.1.3, Namespace monitoring
Grafana          Dashboard "Dispatch City – Betrieb", Events 2.48–2.58/s, offene Bestellungen 5
Rückstau         restaurant-pizza 0 Replicas -> Queue auf 16; replicas=1 -> 0 in < 16 s
Readiness        Pod 0/1 und zurück auf 1/1, restartCount 0, EndpointSlice ready=false
Rollback         nginx:absichtlich-falsch -> ImagePullBackOff, HTTP 200 durchgehend, undo in ~1 s
HPA              min 2, max 4, Ziel 50 %; 2 -> 3 -> 4 -> 2
Endstand         15 Pods in food-delivery 1/1, food-delivery-db healthy 2/2,
                 lab-web nginx:1.28.0-alpine 2/2, restaurant-pizza 1/1
Abweichungen     hpa.yaml maxReplicas 3 -> 4; RabbitMQ-Probe-Werte im Repo korrigiert
```

## Aufräumen

```powershell
# Port-Forwards beenden (Strg+C im jeweiligen Terminal)
kubectl --context k3d-teko-k8s delete namespace betrieb-lab
# Fachsystem und Monitoring bleiben als Endstand bestehen.
```
