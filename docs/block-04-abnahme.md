# Block 4 – Abnahme

Durchgeführt am 17.08.2026 auf demselben k3d-Cluster `teko-k8s` (Kontext `k3d-teko-k8s`),
K3s v1.35.5+k3s1, drei Nodes Ready. Der Namespace `food-delivery` aus Block 3 lief noch,
das Wiederanlaufverfahren aus Aufgabe 2 hätte ihn sonst neu erzeugt.

Ausgangsstand: eigener Projektstand nach Block 3 (Commit `AB3: Deployments, Services und
ConfigMaps – Abnahme`). Darüber installiert: `SwitzerChees/vsc-dispatch-city-04-ingress`,
Tag `v1.1.1` (Commit `692fafc`), geklont nach `../vsc-dispatch-city-04-ingress`.

## Was der Baustein mitbringt

`install.ps1 -Target "."` hat genau drei Dinge in den Projektstand kopiert:

| Pfad | Zweck |
| ---- | ----- |
| `deploy/overlays/block-04-ingress/` | Overlay: `ingress.yaml`, `dashboard-patch.yaml`, `kustomization.yaml` |
| `apps/dashboard/server/routes/ui-instance.get.ts` | Nitro-Route, gibt `HOSTNAME` = Pod-Name zurück |
| `.block-backups/dashboard-index.block-03.vue` | Sicherung der Block-3-Seite |

`apps/dashboard/pages/index.vue` wurde überschrieben, war aber bit-identisch zur Block-3-Fassung
(`diff` leer, `git status` meldet die Datei nicht als geändert). Die Sicherung ist trotzdem
angelegt worden — das Skript sichert unbesehen, bevor es kopiert.

Die Instanz-Auskunft ist bewusst minimal:

```ts
export default defineEventHandler(() => ({
  instance: process.env.HOSTNAME || 'local-dashboard',
}))
```

`HOSTNAME` setzt die Container-Runtime auf den Pod-Namen. Die Route braucht deshalb keine
Kubernetes-API und keine Downward-API — sie liest nur, wie der Pod ohnehin heisst.

## Systemgrenze nach Block 4

| Komponente    | Rolle                                          | Repliken | Erreichbar über |
| ------------- | ---------------------------------------------- | -------- | --------------- |
| `traefik` (kube-system) | Ingress-Controller, einziger HTTP-Eingang | vorinstalliert | `service/traefik`, Typ LoadBalancer |
| `dashboard`   | Nuxt 4 / PixiJS, zustandslos                   | **2**    | Ingress `/` |
| `control-api` | Go, Simulation im Prozessspeicher              | 1        | Ingress `/api`, `/health`, `/metrics` |

Neu gegenüber Block 3: **ein** Einstiegspunkt statt zweier Port-Forwards, und das Dashboard
ist erstmals mehrfach vorhanden.

## Evidenz 1 – gerenderter Stand vor jedem Cluster-Zugriff

`kubectl kustomize ./deploy/overlays/block-04-ingress` rendert **sieben** Ressourcen
(Block 3 waren es sechs; hinzu kommt der Ingress):

```
Ingress   food-delivery      ingressClassName: traefik
Deployment dashboard         replicas: 2
Deployment control-api       replicas: 1
```

Das Overlay setzt zusätzlich das Label `course.teko.ch/block: "04"` mit `includeSelectors: false`
— dieselbe Vorsicht wie in Block 3, damit `spec.selector` unverändert bleibt.

**Nachweis Aufgabe 1:** Ingress-Klasse `traefik`, Dashboard-Replikazahl `2`.

## Evidenz 2 – Bauen, Importieren, Ausrollen

```
food-delivery-control-api:local   7d86576c322d   14.7MB
food-delivery-dashboard:local     0367ff416665   240MB
```

Das Dashboard-Image wurde neu gebaut (die Nitro-Route `ui-instance` liegt als
`.output/server/chunks/routes/ui-instance.get.mjs` im Image). Der Go-Build lief vollständig aus
dem Cache und ergab dieselbe Image-ID wie in Block 3 — die Go-Quellen haben sich nicht geändert.

`k3d image import -c teko-k8s …` → *Successfully imported 2 image(s) into 1 cluster(s)*.
Weiterhin nötig, weil jeder Node seinen eigenen containerd-Store hat.

`kubectl apply -k` meldete sechsmal `configured` und einmal `created` (den Ingress).
Interessant ist, was danach **nicht** passierte:

| Deployment | Pod-Alter danach | Grund |
| ---------- | ---------------- | ----- |
| `control-api` | 5 h 12 min, `RESTARTS 0` | Das Overlay ändert nur `metadata.labels`, nicht das Pod-Template → kein neues ReplicaSet |
| `dashboard` | neu erzeugt | Das Patch ändert `spec.template` (`NUXT_PUBLIC_API_BASE`) → rollierender Ersatz beider Pods |

Derselbe Mechanismus wie beim ConfigMap-Versuch in Block 3: Kubernetes startet Pods neu, wenn
sich der Pod-Hash ändert, und sonst nicht.

**Nachweis Aufgabe 2:** `control-api 1/1`, `dashboard 2/2`, Ingress-Klasse `traefik`.

```
NAME                          READY   UP-TO-DATE   AVAILABLE
deployment.apps/control-api   1/1     1            1
deployment.apps/dashboard     2/2     2            2

NAME            CLASS     HOSTS   ADDRESS                            PORTS
food-delivery   traefik   *       172.20.0.2,172.20.0.3,172.20.0.5   80
```

Die drei Adressen sind die k3d-Node-IPs — `service/traefik` ist vom Typ LoadBalancer und k3s
veröffentlicht ihn über alle Nodes (`80:30765/TCP`).

## Evidenz 3 – ein Einstiegspunkt, drei Routen

Port-Forward `service/traefik 8080:80` (Port 8080 war frei). Alle drei Aufrufe über **dieselbe**
Basisadresse:

| Aufruf | Status | Content-Type | Backend laut Ingress |
| ------ | ------ | ------------ | -------------------- |
| `http://localhost:8080/` | 200 | `text/html;charset=utf-8` | `dashboard:3000` |
| `http://localhost:8080/api/v1/snapshot` | 200 | `application/json` (62 555 Bytes) | `control-api:8080` |
| `http://localhost:8080/health/ready` | 200 | `application/json` (16 Bytes) | `control-api:8080` |

Der Content-Type belegt, dass wirklich zwei verschiedene Dienste antworten und nicht einer
alles ausliefert. Die Regeln stehen so im Ingress:

```
*   /api       control-api:8080
    /health    control-api:8080
    /metrics   control-api:8080
    /          dashboard:3000
```

`pathType: Prefix` und die längste passende Regel gewinnt: `/api/v1/snapshot` trifft `/api`,
nicht `/`. Die Catch-all-Regel `/` steht bewusst zuletzt.

**Nachweis Aufgabe 3:** 200 / 200 / 200.

### Der eigentliche Gewinn: kein zweiter Port-Forward mehr

Block 3 brauchte zwei Port-Forwards, weil das Dashboard-Deployment
`NUXT_PUBLIC_API_BASE=http://localhost:8081` setzte und die API vom **Browser** aufgerufen wird.
Das Block-4-Patch setzt den Wert auf `""`:

```yaml
env:
  - name: NUXT_PUBLIC_API_BASE
    value: ""
```

Damit bildet `useDeliveryApi()` relative URLs, der Browser ruft `/api/v1/…` gegen dieselbe
Origin auf, und der Ingress leitet das intern an `control-api` weiter. Ein einziger Eingang,
und nebenbei kein CORS-Thema mehr.

## Evidenz 4 – Load Balancing sichtbar gemacht

20 Anfragen mit `Connection: close` an `/ui-instance`:

```
10 {"instance":"dashboard-75dbdc8bb6-c97wj"}
10 {"instance":"dashboard-75dbdc8bb6-skd2r"}
```

```
NAME              ADDRESSTYPE   PORTS   ENDPOINTS
dashboard-2rdvn   IPv4          3000    10.42.1.18,10.42.3.14
```

Zwei Pods, zwei bereite Adressen, zwei Nodes (`agent-0` und `agent-1`).

`Connection: close` ist nicht Deko: mit Keep-Alive bleibt eine einzige TCP-Verbindung offen und
alle 20 Antworten kämen vom selben Pod. Die Verteilung passiert beim **Verbindungsaufbau**,
nicht pro HTTP-Anfrage.

Die Aufteilung war in beiden Durchläufen exakt 10 zu 10. Das ist die Signatur von Traefiks
eigenem Round-Robin: Traefik 3.6 läuft hier ohne `nativeLB`, löst den Service also selbst zur
EndpointSlice auf und verteilt der Reihe nach über die Pod-IPs. Ginge der Verkehr über die
ClusterIP, würde kube-proxy per iptables **zufällig** wählen — dann wäre eine exakte
10:10-Teilung zweimal hintereinander unwahrscheinlich.

**Nachweis Aufgabe 4:** `dashboard-75dbdc8bb6-c97wj` und `dashboard-75dbdc8bb6-skd2r`.
Die stabile Rolle des Services: Der Service ist der unveränderliche Name samt ClusterIP, hinter
dem die Menge der bereiten Pod-Adressen ständig wechseln darf — Aufrufer adressieren nie einen
Pod, sondern immer den Service.

### Stolperfalle Windows PowerShell 5.1

Der PowerShell-Einzeiler des Arbeitsblatts setzt PowerShell 7 voraus. Unter Windows PowerShell 5.1
scheitert er 20-mal mit

```
Invoke-RestMethod : Keep-Alive und Close können mit dieser Eigenschaft nicht festgelegt werden.
```

`Connection` ist dort ein geschützter Header und lässt sich nicht über `-Headers` setzen.
Zwei Wege, die funktionieren — beide liefern dieselbe 10:10-Verteilung:

```powershell
# Variante A: HttpWebRequest, KeepAlive explizit aus
1..20 | ForEach-Object {
    $q = [System.Net.HttpWebRequest]::Create("http://localhost:8080/ui-instance")
    $q.KeepAlive = $false
    $resp = $q.GetResponse()
    $t = (New-Object System.IO.StreamReader($resp.GetResponseStream())).ReadToEnd()
    $resp.Close()
    ($t | ConvertFrom-Json).instance
} | Group-Object | Select-Object Name, Count
```

```powershell
# Variante B: curl.exe (ab Windows 10 in system32 vorhanden)
1..20 | ForEach-Object { curl.exe -s -H "Connection: close" http://localhost:8080/ui-instance } |
    Sort-Object -Unique
```

## Evidenz 5 – einen Pod löschen, ohne den Dienst zu verlieren

Gelöscht: `dashboard-75dbdc8bb6-c97wj`. Der Verlauf aus `kubectl get pods -w`:

```
c97wj   1/1   Running             0   115s
c97wj   1/1   Terminating         0   115s
c97wj   0/1   Completed           0   116s
ffzpp   0/1   Pending             0   0s
ffzpp   0/1   ContainerCreating   0   0s
ffzpp   0/1   Running             0   2s
ffzpp   1/1   Running             0   13s
```

Bemerkenswert ist die Reihenfolge: der Ersatz-Pod `ffzpp` wird erst `Pending`, **nachdem** der
alte terminiert — beim `delete pod` sinkt die Ist-Zahl auf 1, und erst darauf reagiert das
ReplicaSet. Das ist der Unterschied zum `rollout restart` aus Block 3, wo die
RollingUpdate-Strategie den neuen Pod zuerst bereitstellt.

Die 20 Anfragen während des Ersatzes:

```
20 × HTTP 200, alle beantwortet von dashboard-75dbdc8bb6-skd2r
Antwortzeiten 0.011 s – 0.047 s, kein 502, kein 503
```

Rund 13 Sekunden lang gab es nur einen bereiten Endpunkt, und genau dafür sind zwei Repliken da.
Dass `ffzpp` in der Ausgabe nicht vorkommt, ist kein Widerspruch: die 20 Anfragen waren nach
etwa 0,3 Sekunden durch, da war der neue Pod noch im `ContainerCreating`. Ein nicht bereiter Pod
steht mit `ready: false` in der EndpointSlice und bekommt keinen Verkehr — dieselbe
Readiness-Mechanik, die in Block 3 an der absichtlich kaputten Probe sichtbar wurde. Dass er
danach in die Rotation kommt, zeigt die EndpointSlice:

```
dashboard-2rdvn   IPv4   3000   10.42.1.18,10.42.3.15
```

Die alte Adresse `10.42.3.14` ist verschwunden, `10.42.3.15` steht an ihrer Stelle. Der
Service-Name blieb dabei unverändert — das ist die ganze Idee.

Gegenprobe nach dem Rollout, nochmals 20 Anfragen:

```
10 {"instance":"dashboard-75dbdc8bb6-ffzpp"}
10 {"instance":"dashboard-75dbdc8bb6-skd2r"}
```

Der Ersatz-Pod trägt wieder die halbe Last, ohne dass irgendwo eine Adresse nachgeführt wurde.

**Nachweis Aufgabe 5:** alt `dashboard-75dbdc8bb6-c97wj`, neu `dashboard-75dbdc8bb6-ffzpp`.
Beobachtung: durchgehend HTTP 200, die zweite Replik trug die Last allein, das Deployment
erreichte nach 13 Sekunden wieder 2/2.

## Was Block 4 nicht löst

Skaliert wurde das **zustandslose** Dashboard. Die Grenze aus Block 3 besteht unverändert:
`control-api` bleibt bei `replicas: 1`, weil der Simulationszustand im Prozessspeicher liegt
(`internal/simulation.Engine`) und die SSE-Streams pod-lokal sind. Ein Ingress verteilt Anfragen,
er repliziert keinen Zustand — zwei API-Repliken würden hinter demselben Ingress genau die
widersprüchlichen Welten zeigen wie in Block 3.

## Abnahmestand

```
deployment.apps/control-api   1/1
deployment.apps/dashboard     2/2
ingress/food-delivery         class traefik   ADDRESS 172.20.0.2,172.20.0.3,172.20.0.5
  /api,/health,/metrics -> control-api:8080
  /                     -> dashboard:3000
endpointslice dashboard-2rdvn   -> 10.42.1.18, 10.42.3.15   (beide ready)
configmap/simulation-config     DATA 2   (APP_MODE, TICK_MS="2000" aus Block 3)
```

## Aufräumen

```powershell
# Port-Forward beenden (Strg+C im jeweiligen Terminal)
kubectl delete namespace food-delivery
# Der Cluster teko-k8s bleibt bestehen.
```
