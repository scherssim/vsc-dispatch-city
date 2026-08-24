# Block 5 – Abnahme

Durchgeführt am 24.08.2026 auf demselben k3d-Cluster `teko-k8s` (Kontext `k3d-teko-k8s`),
K3s v1.35.5+k3s1, drei Nodes Ready. Der Namespace `food-delivery` aus Block 4 lief noch und
wurde in place erweitert — kein Neuaufbau.

Ausgangsstand: eigener Projektstand nach Block 4 (Commit `AB4: Ingress und externe Zugriffe –
Abnahme`). Darüber installiert: `SwitzerChees/vsc-dispatch-city-05-messaging`, Tag `v1.1.0`
(Commit `eef2d40`), geklont nach `../vsc-dispatch-city-05-messaging`.

Alle fünf Aufgaben des Arbeitsblatts sind durchgeführt.

## Was der Baustein mitbringt

Anders als in Block 4 kopiert `install.sh` nicht drei Dateien, sondern ersetzt den halben
Go-Quellbaum. `git status` nach der Installation:

| Kategorie | Pfade |
| --------- | ----- |
| Geändert | `cmd/control-api/main.go`, `internal/api/server.go`, `go.mod`, `go.sum`, `scripts/build-images.sh`, `scripts/load-images.sh`, `docs/architecture.md` |
| Neu | `cmd/{customer-simulator,restaurant-worker,courier-simulator,order-worker}`, `internal/{messaging,appenv,telemetry,workerutil}`, `internal/api/commands.go`, `deploy/overlays/block-05-messaging/`, `scripts/lab-publish.{sh,ps1}`, `scripts/{build,load}-images.ps1` |

Unverändert blieben `internal/simulation`, `internal/model`, `internal/events` und
`internal/scenario` — die Simulationsmechanik ist dieselbe wie in Block 3, sie wird nur nicht
mehr in einem Prozess ausgeführt.

Nebeneffekt, der eine Lücke aus Block 4 schliesst: das mitgelieferte `docs/architecture.md`
enthält den Abschnitt „Block 4: Ingress und Load Balancing", der beim Block-4-Baustein nicht
mitkopiert worden war, plus den neuen Abschnitt „Block 5: Messaging".

## Systemgrenze nach Block 5

| Komponente | Rolle | Repliken | Queue |
| ---------- | ----- | -------- | ----- |
| `rabbitmq` | Broker, StatefulSet mit 1-GiB-PVC | 1 | — |
| `control-api` | REST/SSE, publiziert und konsumiert | 1 | `live.<pod>` (exklusiv, auto-delete) |
| `customer-simulator` | Producer, StatefulSet | 1 | `simulation-control.<pod>` |
| `restaurant-pizza/-bowl/-curry` | Consumer, je eigene Queue | je 1 | `restaurant.<id>` |
| `courier-simulator` | Consumer + Producer, StatefulSet | 1 | `courier-dispatch` |
| `order-worker` | Projektion, Competing Consumers | 2 | `order-projection` |
| `dashboard` | Nuxt/PixiJS, zustandslos | 2 | — |

Neu gegenüber Block 4: Die Control API ist nicht mehr die Simulation, sondern nur noch deren
Fenster. Sie hält weiterhin `replicas: 1`, weil der projizierte Zustand im Prozessspeicher liegt
— die Persistenz kommt erst in Block 6.

## Aufgabe 1 – Baustein integrieren und Topologie rendern

`kubectl kustomize ./deploy/overlays/block-05-messaging` rendert **22 Ressourcen** (Block 4
waren es sieben). Das Overlay baut auf `../block-04-ingress` auf und ergänzt `rabbitmq.yaml`,
`workers.yaml` sowie zwei Patches.

**Nachweis Aufgabe 1:**

```
Betriebsmodus   ConfigMap simulation-config -> APP_MODE: distributed   (Patch aus Block 5)
Broker-Image    rabbitmq:4.3.5-management-alpine
Worker          6 fachliche Workloads in 4 Rollen:
                  customer-simulator     StatefulSet   food-delivery-customer-simulator:local
                  restaurant-pizza       Deployment  \
                  restaurant-bowl        Deployment   > food-delivery-restaurant-worker:local
                  restaurant-curry       Deployment  /
                  courier-simulator      StatefulSet   food-delivery-courier-simulator:local
                  order-worker           Deployment    food-delivery-order-worker:local
PVC             volumeClaimTemplates: data, 1Gi, ReadWriteOnce
```

Das Overlay setzt `course.teko.ch/block: "05"` mit `includeSelectors: false` — dieselbe Vorsicht
wie in Block 3 und 4, damit `spec.selector` der bestehenden Deployments unverändert bleibt und
`kubectl apply` sie nicht ablehnt.

Warum drei Restaurant-Deployments statt drei Repliken eines Deployments: jeder Worker konsumiert
nur seine eigene Queue `restaurant.<id>`, gebunden an den Routing Key seines Restaurants. Das ist
Sharding nach fachlichem Schlüssel, nicht Competing Consumers — die kommen in Aufgabe 4 innerhalb
*einer* Restaurant-Queue.

## Aufgabe 2 – Bauen, Importieren, Ausrollen

`go test ./...` grün (`internal/scenario` 1.9s, `internal/simulation` 2.1s). `-race` liess sich
nicht ausführen: der Detektor braucht cgo, und auf dieser Windows-Installation fehlt der
C-Compiler. Ohne Detektor ist der Test aussagekräftig für die Logik, nicht für Datenrennen.

Sechs Images gebaut und importiert:

```
food-delivery-control-api:local          50245e4d7cea   15.8MB
food-delivery-customer-simulator:local   58d70fd884ef   15.7MB
food-delivery-restaurant-worker:local    1f3c41467cea   15.7MB
food-delivery-courier-simulator:local    bc34db3cebea   15.7MB
food-delivery-order-worker:local         595ae0e3197f   15.6MB
food-delivery-dashboard:local            64a2c5380986    240MB
```

Die fünf Go-Images teilen sich dasselbe `build/go-service.Dockerfile`; der Service kommt über
`--build-arg SERVICE`. Alle liegen bei rund 15.7 MB, weil sie dieselbe Basis und dieselbe
Bibliotheksmenge tragen und sich nur im einkompilierten `main` unterscheiden.
`k3d image import -c teko-k8s …` → *Successfully imported 6 image(s) into 1 cluster(s)*.

`kubectl apply -k` meldete 12× `created`, 9× `configured`, 1× `unchanged`.

### Stolperfalle: RabbitMQ im Restart-Loop

Der erste `rollout status statefulset/rabbitmq --timeout=300s` lief in den Timeout. `rabbitmq-0`
stand nach fünf Minuten bei vier Restarts:

```
Warning  Unhealthy  Readiness probe failed: command timed out:
                    "rabbitmq-diagnostics -q check_running" timed out after 1s
Warning  Unhealthy  Liveness probe failed: command timed out:
                    "rabbitmq-diagnostics -q ping" timed out after 1s
Normal   Killing    Container rabbitmq failed liveness probe, will be restarted
```

Beide Probes im gelieferten `rabbitmq.yaml` definieren `initialDelaySeconds` und
`periodSeconds`, aber kein `timeoutSeconds` — Kubernetes nimmt dann den Default von **einer**
Sekunde. `rabbitmq-diagnostics` ist ein Erlang-CLI, das für jeden Aufruf eine eigene VM startet
und sich an den Broker-Node anhängt; unter einer Sekunde ist das auf diesem Laptop nicht zu
schaffen. Die Liveness-Probe hat den Broker also getötet, bevor er je fertig booten konnte —
ein Deadlock aus Selbstheilung.

Angepasst in `deploy/overlays/block-05-messaging/rabbitmq.yaml`:

| Probe | vorher | nachher |
| ----- | ------ | ------- |
| readiness | `initialDelay 8`, `period 5`, timeout implizit 1 | `initialDelay 20`, `period 15`, `timeout 10`, `failureThreshold 6` |
| liveness | `initialDelay 20`, `period 10`, timeout implizit 1 | `initialDelay 60`, `period 30`, `timeout 15`, `failureThreshold 3` |

Danach war `rabbitmq-0` nach 82 Sekunden ohne Restart `1/1`. Die Anpassung ist eine bewusste
Abweichung vom Kurspaket und würde bei einer erneuten Ausführung von `install.sh` überschrieben.

Der Restart-Zähler der übrigen Pods ist ein Folgeschaden desselben Problems und kein zweiter
Fehler — sie warten aktiv auf den Broker und geben irgendwann auf:

```
{"level":"WARN","msg":"waiting for RabbitMQ","attempt":30,
 "error":"dial tcp 10.43.4.121:5672: connect: connection refused"}
{"level":"ERROR","msg":"create RabbitMQ publisher","error":"connect to RabbitMQ: context canceled"}
```

Das ist korrektes Verhalten für einen Consumer ohne Broker: lieber sterben und vom
ReplicaSet neu gestartet werden, als in einem halb funktionsfähigen Zustand weiterlaufen. Sobald
RabbitMQ `1/1` war, liefen alle Workloads ohne weiteren Restart.

**Nachweis Aufgabe 2:**

```
statefulset.apps/rabbitmq             1/1     (rabbitmq-0, 0 Restarts nach Probe-Fix)
deployment.apps/dashboard             2/2
deployment.apps/order-worker          2/2
deployment.apps/control-api           1/1
deployment.apps/restaurant-pizza      1/1
deployment.apps/restaurant-bowl       1/1
deployment.apps/restaurant-curry      1/1
statefulset.apps/customer-simulator   1/1
statefulset.apps/courier-simulator    1/1
pvc/data-rabbitmq-0                   Bound   pvc-33f7e32e…   1Gi   RWO   local-path
configmap/simulation-config           {"APP_MODE":"distributed","TICK_MS":"2000"}
```

## Aufgabe 3 – Eventfluss verfolgen

Zwei Port-Forwards, wie im Arbeitsblatt: `service/traefik 8080:80` in `kube-system` und
`service/rabbitmq 15672:15672` in `food-delivery`.

`GET /api/v1/snapshot` über den Ingress bestätigt den Moduswechsel — dasselbe Feld, aus dem das
Dashboard sein Badge zieht:

```json
{"mode":"distributed","running":true,"instance":"control-api-6c9598cb5c-c9bnp", …}
```

RabbitMQ Management meldet `RabbitMQ 4.3.5 | Erlang 27.3.4.16 | node rabbit@rabbitmq-0`.

### Topologie zur Laufzeit

Zwei Topic-Exchanges und acht Queues — genau der erwartete Stand:

```
exchange  food.events   topic     Produktivpfad
exchange  food.dlx      topic     Dead Letter, gebunden mit '#'

name                                       durable  consumers  messages_ready
live.control-api-6c9598cb5c-c9bnp          false        1           0
food.dead                                  true         0           0
simulation-control.customer-simulator-0    true         1           0
courier-dispatch                           true         1           0
order-projection                           true         2           0
restaurant.restaurant-pizza                true         1           0
restaurant.restaurant-curry                true         1           0
restaurant.restaurant-bowl                 true         1           0
```

Drei Dinge sind daran ablesbar:

- `order-projection` hat **zwei** Consumer bei einer Queue — die beiden `order-worker`-Pods sind
  bereits Competing Consumers, ohne dass etwas skaliert werden musste.
- `live.<pod>` ist als einzige Queue `durable=false` und trägt den Pod-Namen. Sie wird pro
  API-Pod exklusiv angelegt und mit dem Pod wieder weggeräumt (`Exclusive`, `AutoDelete` in
  `cmd/control-api/main.go`). Eine SSE-Ansicht überlebt ihren Pod bewusst nicht.
- Jede fachliche Queue wird mit `x-dead-letter-exchange: food.dlx` deklariert
  (`internal/messaging/rabbitmq.go:117`). Der Weg in die DLQ ist also schon gebaut, bevor
  Aufgabe 5 ihn benutzt.

### Beobachtete Eventkette

Bestellung manuell über den Ingress ausgelöst: `POST /api/v1/orders` →
`201 Created`, Order `4d68b745-d0fb-48da-9f90-5bdf6e530029`, Kunde `customer-agent-10001`,
Restaurant `restaurant-bowl`.

| Zeit (UTC) | Station | Beleg |
| ---------- | ------- | ----- |
| 17:58:25.403 | Control API publiziert `order.created` nach `food.events` | HTTP 201, `status: created` |
| 17:58:26.312 | `restaurant-bowl` konsumiert aus `restaurant.restaurant-bowl`, antwortet mit `order.accepted` | `{"msg":"order processed","order_id":"4d68b745…","result":"order.accepted"}` |
| ~17:58:28 | `courier-simulator` konsumiert `order.accepted` aus `courier-dispatch`, weist `courier-1` zu | Snapshot-Status `courier_to_restaurant` |
| ~17:58:34 | Abholung, Fahrt zum Kunden | Snapshot-Status `in_transit` |
| 17:58:45.025 | `courier-simulator` publiziert `order.delivered` | `{"msg":"order delivered","order_id":"4d68b745…","courier_id":"courier-1"}` |

Statusverlauf aus dem Snapshot, im Drei-Sekunden-Takt abgefragt:
`created → accepted → courier_to_restaurant → in_transit → delivered`, Gesamtdauer rund
20 Sekunden.

Die Kette lief über drei Pods, die einander nicht kennen. Weder die Control API noch der
Restaurant-Worker haben je den Kurier aufgerufen — jeder Schritt ist ein Event auf
`food.events`, das der nächste Consumer über seinen Routing Key aufgreift. Genau das ist der
Unterschied zu Block 4, wo dieselbe Kette als Methodenaufrufe innerhalb eines Prozesses lief.

Parallel dazu produziert `customer-simulator-0` alle 15 Sekunden von sich aus Bestellungen
(`ORDER_INTERVAL_MS: 15000`), im Wechsel über alle drei Restaurants:

```
17:58:05  order published  bfc1b1e9…  restaurant-pizza
17:58:20  order published  f12c25e4…  restaurant-bowl
17:58:35  order published  032d8adc…  restaurant-curry
```

`/metrics` nach dem Durchlauf: `food_delivery_delivered_orders_total 8`,
`food_delivery_events_total 269`, `food_delivery_active_orders 2`.

## Aufgabe 4 – Rückstau erzeugen und mit Competing Consumers abbauen

### Vorbereitung: die Messung isolieren

Vor dem Skalieren habe ich die Simulation über `POST /api/v1/simulation/pause` angehalten. Grund:
`customer-simulator-0` produziert alle 15 Sekunden im Wechsel über die drei Restaurants
selbstständig Bestellungen, also rund alle 45 Sekunden eine Pizza-Order. Ohne Pause wäre der
gemessene Rückstau nicht die publizierte Acht, sondern acht plus Drift. Der Simulator hört auf
`simulation.paused` und stellt sein Publizieren ein (`cmd/customer-simulator/main.go:61`) — die
Pause wirkt also über denselben Eventweg, den die Aufgabe untersucht, und nicht per Eingriff von
aussen. Das ist eine Ergänzung zum Arbeitsblatt zugunsten eines reproduzierbaren Messwerts.

### Rückstau ohne Consumer

```
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=0
kubectl -n food-delivery wait --for=delete pod -l app.kubernetes.io/instance=restaurant-pizza
./scripts/lab-publish.sh valid 8
```

```
name                          consumers  messages_ready
restaurant.restaurant-pizza        0            8
order-projection                   2            0
food.dead                          0            0
```

Exakt acht Nachrichten auf `ready`, kein Consumer. Die Queue ist `durable`, die Nachrichten
werden mit `delivery_mode: 2` publiziert — der Rückstau liegt auf dem PVC und überlebt einen
Broker-Neustart.

Aufschlussreich ist die Zeile darunter: `order-projection` steht bei **0**, obwohl dieselben acht
Events auch dort gelandet sind. Der Routing Key `order.created.restaurant-pizza` passt auf die
Bindung `order.#` des Order Workers. Dessen zwei Pods liefen weiter und haben ihre Kopien sofort
verarbeitet. Ein Consumer-Ausfall staut also nur seine eigene Queue — die übrigen Konsumenten
desselben Events merken davon nichts. Genau das ist die Entkopplung, die ein synchroner
HTTP-Aufruf nicht liefert.

### Abbau mit zwei Consumern

```
kubectl -n food-delivery scale deployment/restaurant-pizza --replicas=2
```

```
name                          consumers  messages_ready
restaurant.restaurant-pizza        2            0
```

Die Logs beider Pods zeigen striktes Round Robin:

| Zeit (UTC) | Pod | Order |
| ---------- | --- | ----- |
| 18:16:35.944 | `…-qvc2q` | `lab-pizza-…-1` |
| 18:16:36.098 | `…-ml2rn` | `lab-pizza-…-2` |
| 18:16:36.859 | `…-qvc2q` | `lab-pizza-…-3` |
| 18:16:37.022 | `…-ml2rn` | `lab-pizza-…-4` |
| 18:16:37.791 | `…-qvc2q` | `lab-pizza-…-5` |
| 18:16:37.941 | `…-ml2rn` | `lab-pizza-…-6` |
| 18:16:38.709 | `…-qvc2q` | `lab-pizza-…-7` |
| 18:16:38.873 | `…-ml2rn` | `lab-pizza-…-8` |

**Nachweis Aufgabe 4:** vorher `consumers 0 / messages_ready 8`, nachher `consumers 2 /
messages_ready 0`. Beteiligte Pods: `restaurant-pizza-6b7d8c5486-qvc2q` (ungerade Nummern) und
`restaurant-pizza-6b7d8c5486-ml2rn` (gerade Nummern). Acht Nachrichten in 2.93 Sekunden.

Die saubere Abwechslung ist kein Zufall, sondern `Prefetch: 1` in
`cmd/restaurant-worker/main.go`: jeder Pod bekommt genau eine unbestätigte Nachricht, die
nächste erst nach dem Ack. Mit hohem Prefetch hätte ein Pod den halben Rückstau vorab in seinen
lokalen Puffer gezogen und der zweite hätte kaum etwas zu tun bekommen — sichtbar wäre dann
Skalierung ohne Wirkung.

### Nebenbefund: der Kurier ist der eigentliche Engpass

Nach dem Abbau standen fünf Nachrichten auf `courier-dispatch`. `courier-simulator-0` ist eine
einzelne Replik und braucht pro Auslieferung rund 20 Sekunden Fahrzeit; acht gleichzeitig
angenommene Bestellungen kann er nur nacheinander abarbeiten. Der Restaurant-Rückstau war in drei
Sekunden weg, der Kurier-Rückstau nicht — die Queue macht den Engpass sichtbar, statt ihn wie ein
synchroner Aufruf in Timeouts zu verstecken.

Zweite Beobachtung dabei: `courier-simulator` wertet `simulation.paused` nicht aus — anders als
der Customer-Simulator hat er keine Behandlung dafür. Er hat während der pausierten Simulation
weiter ausgeliefert. Fachlich ist das inkonsistent; für diesen Block ohne Folgen, aber es ist der
Grund, warum die Pause nur die Produktion und nicht die Verarbeitung stoppt.

## Aufgabe 5 – Ungültige Nachricht in der Dead Letter Queue isolieren

```
kubectl -n food-delivery exec rabbitmq-0 -- rabbitmqctl purge_queue food.dead
./scripts/lab-publish.sh invalid
```

Publiziert wird der Payload `this-is-not-json` mit dem regulären Routing Key
`order.created.restaurant-pizza` — eine syntaktisch gültige AMQP-Nachricht mit fachlich
unlesbarem Inhalt.

```
name                          messages_ready
food.dead                           3
restaurant.restaurant-pizza         0
order-projection                    0
```

**Nachweis Aufgabe 5:** `food.dead` enthält **drei** Kopien. Der `x-death`-Header jeder Kopie
nennt ihre Herkunft:

| Herkunftsqueue | reason | count | Bindung, über die die Kopie entstand |
| -------------- | ------ | ----- | ------------------------------------ |
| `restaurant.restaurant-pizza` | rejected | 1 | Routing Key des Restaurants |
| `order-projection` | rejected | 1 | `order.#` |
| `live.control-api-6c9598cb5c-c9bnp` | rejected | 1 | `#` |

### Warum dieselbe Publikation mehrfach dead-lettered wird

Weil es nicht dieselbe Nachricht ist. `food.events` ist ein **Topic Exchange**: er liefert eine
Publikation an *jede* Queue, deren Bindung auf den Routing Key passt. Drei Bindungen passen auf
`order.created.restaurant-pizza`, also entstehen im Broker drei eigenständige Kopien mit je
eigenem Ack-Lebenszyklus. Alle drei Consumer scheitern am selben `json.Unmarshal`, jeder
quittiert seine Kopie mit einem Reject ohne Requeue, und jede Queue trägt
`x-dead-letter-exchange: food.dlx` (`internal/messaging/rabbitmq.go:117`). Damit wandern drei
Kopien über `food.dlx` in die mit `#` gebundene `food.dead`.

Der Beleg dafür, dass es Fan-out und keine Wiederholschleife ist, steht in `count: 1`: jede Kopie
wurde genau einmal dead-lettered. Ein Retry-Loop würde stattdessen eine Kopie mit steigendem
Zähler zeigen. Die Zahl der DLQ-Einträge ist also keine Fehlerhäufigkeit, sondern die Anzahl der
Konsumenten, die dieses Event fachlich interessiert hat.

Dass die Kopie in `live.<pod>` denselben Weg geht, ist ein Nebeneffekt der `#`-Bindung: der
SSE-Stream des Dashboards abonniert bewusst alles und stolpert deshalb über jede kaputte
Nachricht mit.

### Die regulären Queues bleiben verarbeitbar

Alle fachlichen Queues stehen nach dem Vorfall wieder bei `messages_ready 0`, die Consumer sind
verbunden, und `restaurant-pizza` ist auf `1/1` zurückgesetzt. Die vergiftete Nachricht hat keine
Queue blockiert — sie wurde beim ersten Zustellversuch aussortiert, statt endlos requeued zu
werden und den Kopf der Queue zu belegen. Das ist der Unterschied zwischen `nack(requeue=false)`
mit DLX und einem naiven Retry.

Die drei Kopien liegen weiterhin in `food.dead` und werden von niemandem konsumiert
(`consumers 0`). Sie sind Beweismittel, kein Betriebszustand — in einem echten System hinge dort
ein Alarm oder ein Wiedereinspielungs-Werkzeug.

## Was Block 5 nicht löst

Der Zustand ist verteilt, aber nicht haltbar. Der `order-worker` besitzt nur einen lokalen
Idempotenzspeicher; ein Pod-Neustart verliert ihn, und bei At-least-once-Zustellung kann
dieselbe Nachricht danach ein zweites Mal wirken. Der Broker selbst ist ein einzelner Pod: PVC
und durable Queues überstehen einen Container-Neustart, aber nicht den Ausfall des Nodes. Beides
ist Absicht und Stoff für Block 6.

## Abnahmestand

```
Overlay          deploy/overlays/block-05-messaging   22 Ressourcen
APP_MODE         distributed
Broker           rabbitmq:4.3.5-management-alpine, StatefulSet 1/1, PVC 1Gi Bound
Exchanges        food.events (topic), food.dlx (topic)
Queues           8, davon 7 durable
Eventkette       created → accepted → courier_to_restaurant → in_transit → delivered
Rückstau         0 Consumer -> 8 ready; 2 Consumer -> 0 ready in 2.93 s, striktes Round Robin
DLQ              food.dead 3 Kopien, alle reason=rejected count=1
Endstand         restaurant-pizza 1/1, Simulation läuft, alle fachlichen Queues bei 0
Abweichungen     Probe-Timeouts in rabbitmq.yaml erhöht (Aufgabe 2);
                 Simulation für die Messung in Aufgabe 4 pausiert
```

## Aufräumen

```powershell
# Port-Forwards beenden (Strg+C im jeweiligen Terminal)
kubectl delete namespace food-delivery
# Der Cluster teko-k8s bleibt bestehen.
# Das PVC data-rabbitmq-0 verschwindet mit dem Namespace, der Broker-Zustand also auch.
```
