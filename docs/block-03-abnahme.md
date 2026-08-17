# Block 3 – Abnahme

Durchgeführt am 17.08.2026 auf dem k3d-Cluster `teko-k8s` (Kontext `k3d-teko-k8s`),
Node-Version K3s v1.35.5+k3s1, Runtime containerd 2.2.3-k3s1, drei Nodes Ready
(`server-0` als control-plane, `agent-0`, `agent-1`).

Ausgangsstand: Repository aus dem Template `SwitzerChees/vsc-dispatch-city-03-foundation`,
Release `v1.0.0`. `main` und Tag `v1.0.0` zeigen im Template auf denselben Commit `8de302d`;
`git diff --stat v1.0.0 HEAD` ist leer, der Inhalt ist also bit-identisch zum verbindlichen Stand.

## Systemgrenze

| Komponente    | Rolle                                                                 | Zustand      | Repliken |
| ------------- | --------------------------------------------------------------------- | ------------ | -------- |
| `dashboard`   | Nuxt 4 / PixiJS, zeichnet die 21×21-Tile-Stadt, Port 3000             | zustandslos  | 1 (skalierbar) |
| `control-api` | Go, Simulation + REST + SSE + Health + Metrics, Port 8080             | In-Memory    | 1 (bewusst) |

Die Control API ist die einzige Quelle der Wahrheit. Das Dashboard liest per
`/api/v1/snapshot` und hält über `/api/v1/events` (SSE) nach.

## Evidenz 1 – Image und Deployment

```
food-delivery-control-api   local   806a0cf5c891   14.7MB
food-delivery-dashboard     local   a838925723f5   240MB
```

* `food-delivery-control-api:local` ← `build/go-service.Dockerfile`, `--build-arg SERVICE=control-api`,
  Kontext = Repo-Wurzel. Multi-Stage `golang:1.25-alpine` → `gcr.io/distroless/static-debian12:nonroot`.
* `food-delivery-dashboard:local` ← `apps/dashboard/Dockerfile`, Kontext = `apps/dashboard`.
  `npm ci` → `npm run build` → `node:24-alpine` mit nur `.output`.
* `k3d image import -c teko-k8s …` → *Successfully imported 2 image(s) into 1 cluster(s)*.
  Nötig, weil jeder Node einen eigenen containerd-Store hat und den Docker-Daemon des Hosts nicht sieht.
* `imagePullPolicy: IfNotPresent` ist zwingend: die Tags `:local` existieren in keiner Registry.
  Mit `Always` endeten die Pods in `ImagePullBackOff`.

```
deployment.apps/control-api   1/1   food-delivery-control-api:local   app.kubernetes.io/name=control-api
deployment.apps/dashboard     1/1   food-delivery-dashboard:local     app.kubernetes.io/name=dashboard
```

`kubectl kustomize deploy/overlays/block-03-standalone` rendert genau sechs Ressourcen — alle aus
der Base. Das Overlay fügt keine eigene Ressource hinzu, sondern nur das Label
`course.teko.ch/block: "03"`. Wegen `includeSelectors: false` landet es ausschliesslich in
`metadata.labels`, nicht in `spec.selector` und nicht in den Pod-Template-Labels — wichtig, weil
`spec.selector` eines bestehenden Deployments unveränderlich ist.

## Evidenz 2 – Service, EndpointSlice und DNS

Kette für `control-api`:

```
Service-Selector   app.kubernetes.io/name=control-api
Pod-Labels         app.kubernetes.io/name=control-api, app.kubernetes.io/part-of=food-delivery, pod-template-hash=…
EndpointSlice      control-api-b6k95 → addresses [10.42.1.17], ready: true, targetRef Pod control-api-…
Ports              Service 8080 → targetPort: http → containerPort 8080 (benannter Port)
```

Aus einem Curl-Pod im Namespace:

| Aufruf | Ergebnis |
| ------ | -------- |
| `http://control-api:8080/health/ready` | HTTP 200 `{"status":"ok"}` über 10.43.156.90 |
| `http://control-api.food-delivery.svc.cluster.local:8080/health/ready` | HTTP 200, identische ClusterIP |
| `http://dashboard:3000/` | HTTP 200 über 10.43.4.172 |

Der Kurzname genügt wegen `/etc/resolv.conf` im Pod:

```
search food-delivery.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.43.0.10
options ndots:5
```

Vom Host aus schlägt beides fehl: `Resolve-DnsName control-api.food-delivery.svc.cluster.local`
meldet *Der DNS-Name ist nicht vorhanden* (der Host nutzt nicht CoreDNS), und ein Zugriff auf die
ClusterIP `10.43.156.90:8080` läuft in den Timeout (die ClusterIP existiert nur als
kube-proxy-Regel auf den Nodes). Deshalb ist `port-forward` nötig.

### Readiness-Probe bewusst zum Fehlschlagen gebracht

Probe-Pfad temporär auf `/health/gibtsnicht` gepatcht:

* Neuer Pod `Running`, aber dauerhaft `0/1 READY`; Event
  *Readiness probe failed: HTTP probe failed with statuscode: 404*.
* Er steht in der EndpointSlice, aber mit `ready: false` → kube-proxy leitet nichts dorthin.
* Rollout blockiert (*1 old replicas are pending termination*, `rollout status` Exit 1), weil der
  neue Pod nie ready wird — der alte bereite Pod bediente ununterbrochen weiter.

Readiness steuert also Verkehr und schützt den Rollout; Liveness hätte stattdessen den Container
neu gestartet (CrashLoopBackOff). Zurückgesetzt per `kubectl apply -k`.

## Evidenz 3 – ConfigMap, Probes und Rollout

`TICK_MS` in `deploy/base/configmap.yaml` von `"500"` auf `"2000"` geändert.

1. `kubectl diff -k` zeigt **nur** die ConfigMap: `-TICK_MS: "500"` / `+TICK_MS: "2000"`.
   Die Deployments sind im Diff nicht enthalten.
2. `kubectl apply -k` → `configmap/simulation-config configured`,
   `deployment.apps/control-api unchanged`.
3. Der bestehende Pod lief unverändert weiter (`RESTARTS 0`), Startlog blieb
   `{"msg":"control API started","mode":"standalone","interval":"500ms"}` —
   `envFrom`-Werte werden einmalig beim Containerstart in die Prozessumgebung kopiert.
4. `kubectl rollout restart deployment/control-api` setzt die Annotation
   `kubectl.kubernetes.io/restartedAt`, erzeugt ein neues ReplicaSet und ersetzt den Pod.
   Startlog des neuen Pods: `{"msg":"control API started","mode":"standalone","interval":"2s"}`.
5. Gegenprobe über die Fachlogik: das Feld `tick` im Snapshot stieg über 4 Sekunden um genau 2
   → 2 s pro Tick.

## Sichtbarer Ablauf im Dashboard

Port-Forwards: `service/dashboard 3000:3000` und `service/control-api 8081:8080`.
Der Port 8081 ist nicht frei wählbar — das Dashboard-Deployment setzt
`NUXT_PUBLIC_API_BASE=http://localhost:8081`, und die API wird vom **Browser** aufgerufen,
nicht vom Dashboard-Pod.

Lebenszyklus einer Bestellung, Tick für Tick über `/api/v1/snapshot` verfolgt:

```
tick   Kurier          Position    Bestellung               progress
46     idle            12.0/8.0    (keine)                  -
52     to_restaurant   12.0/8.0    courier_to_restaurant    0.00
60     to_restaurant   7.0/8.0     courier_to_restaurant    0.20
66     to_restaurant   4.0/7.4     courier_to_restaurant    0.35
72     picking_up      4.0/4.0     picked_up                0.50
78     to_customer     6.5/4.0     in_transit               0.60
```

`idle → created → accepted → courier_to_restaurant → picked_up → in_transit → delivered → idle`.
Nach der Auslieferung bleibt der Kurier als freie Flottenentität an seiner letzten Position stehen.

Die Bewegung folgt dem Strassenraster, nicht der Luftlinie: erst ändert sich `x` bei konstantem `y`
(12.0/8.0 → 4.0/8.0), dann `y` bei konstantem `x` (4.0/8.0 → 4.0/4.0). Keine Diagonalen,
keine Teleportation.

SSE-Mitschnitt über 12 Sekunden: 1× `order.created`, 1× `order.accepted`, 1× `courier.assigned`,
1× `order.picked_up`, 16× `courier.location.updated`.

## Architekturgrenze: warum `replicas: 1`

Kurzzeitig auf zwei Repliken skaliert — beide `ready`, aber mit völlig getrennten Welten:

| Pod      | tick | delivered_orders_total | events_total |
| -------- | ---- | ---------------------- | ------------ |
| `…65pwr` | 111  | 6                      | 366          |
| `…b5f84` | 20   | 0                      | 24           |

Über den Service abgefragt antwortete abwechselnd die eine oder die andere Instanz
(Round-Robin über die EndpointSlice, erkennbar am Feld `instance` aus `POD_NAME`). Ein Client sieht
die Stadt zwischen zwei widersprüchlichen Zuständen hin- und herspringen.

Ursache: der Zustand liegt im Prozessspeicher (`internal/simulation.Engine`), die Simulationsschleife
läuft pro Pod eigenständig, und die SSE-Streams sind pod-lokal. Kein gemeinsamer Speicher,
keine Leader-Wahl, keine Idempotenz. Anschliessend per `kubectl apply -k` auf `replicas: 1` zurück.

Was die Grenze später auflöst:

* **Persistenz** (z. B. PostgreSQL) statt Prozessspeicher, damit alle Repliken dieselbe Wahrheit lesen.
* **Messaging / Event-Bus** (Kafka, NATS), damit Domain-Events zwischen Instanzen fliessen.
* **Trennung** in zustandslose API-Repliken und einen einzelnen bzw. per Leader-Election bestimmten
  Simulations-Worker.
* **StatefulSet / Sharding / verteilter Cache**, falls der Zustand doch im Dienst bleiben soll.

Das Dashboard ist bereits heute zustandslos und damit sofort skalierbar.

## Abnahmestand

```
deployment.apps/control-api   1/1
deployment.apps/dashboard     1/1
service/control-api   ClusterIP 10.43.156.90   8080/TCP
service/dashboard     ClusterIP 10.43.4.172    3000/TCP
endpointslice control-api-b6k95   →  10.42.1.17   (ready)
endpointslice dashboard-2rdvn     →  10.42.1.14   (ready)
configmap/simulation-config   DATA 2   (APP_MODE, TICK_MS)
```

## Aufräumen

```bash
kubectl delete namespace food-delivery
# Der Cluster teko-k8s bleibt für Block 4 bestehen.
```
