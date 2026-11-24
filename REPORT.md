# Практична 10. Kubernetes, практична частина

Fork: https://github.com/JuliaVushyvana/k8s-demo

| Рівень | Хто | Розділ |
|---|---|---|
| 60, 75, 90 | Розробник 1 | нижче |
| 100 (StudySwap у Kubernetes) | Розробник 2 | [розділ 100](#100-балів-studyswap-у-kubernetes) |

Середовище: macOS (arm64), Docker Desktop 29.9.3, Node.js 24, kubectl 1.36, Minikube 1.39 (Kubernetes 1.37, runtime containerd).

---

## 60 балів

### 1. Локальний запуск

```bash
git clone https://github.com/JuliaVushyvana/k8s-demo.git
cd k8s-demo
npm install
npm run dev
```

| Ендпоінт | Відповідь | Що повертає |
|---|---|---|
| `GET /api/status` | `{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T13:03:42.219Z"}` | статус, режим із `APP_MODE`, версію і поточний час |
| `GET /api/items?filter=a` | `[{"id":1,"name":"Item A"}]` | статичний список з фільтром за підрядком у `name` (без `filter` — усі 3 елементи) |
| `GET /api/slow` | `{"status":"ok","delayMs":2000}` (через 2002 мс) | відповідь із затримкою `SLOW_MS` (за замовчуванням 2000 мс) або `?ms=` |
| `GET /api/error` | `{"status":"ok","message":"No error"}` | імітація збою: при `ERROR_MODE=always` — 500, `probabilistic` — 500 у ~50 %, `off` — 200 |
| `GET /healthz` | `ok` | liveness: процес живий, завжди 200 |
| `GET /ready` | `{"status":"not-ready"}` **503** у перші 5 с, далі `{"status":"ready"}` 200 | readiness: 503, доки не мине `READINESS_DELAY_MS` |

Лог сервісу:

```
App started on port 8080 in mode dev
[2026-11-24T13:03:37.179Z] GET /ready 503 5ms
[2026-11-24T13:03:42.220Z] GET /api/status 200 1ms
[2026-11-24T13:03:42.240Z] GET /api/items?filter=a 200 1ms
[2026-11-24T13:03:44.257Z] GET /api/slow 200 2002ms
[2026-11-24T13:03:44.293Z] GET /api/error 200 1ms
[2026-11-24T13:03:44.312Z] GET /healthz 200 1ms
[2026-11-24T13:03:44.328Z] GET /ready 200 1ms
```

> `npm install` спершу падав з `EACCES` на `~/.npm/_cacache` (частину кешу колись створено від root). Залежності встановлено з окремим кешем `npm install --cache <тимчасова тека>`; постійне виправлення — `sudo chown -R $(whoami) ~/.npm`.

### 2. Контейнеризація

```bash
docker build -t k8s-demo-app:1.0.0 .
docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
```

```
$ curl http://localhost:8080/api/status
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T13:04:14.673Z"}
$ curl http://localhost:8080/healthz
ok
$ docker ps
k8s-demo-local   k8s-demo-app:1.0.0   Up 2 seconds   0.0.0.0:8080->8080/tcp
```

### 3. Minikube і контекст kubectl

```bash
minikube start --memory=3072 --cpus=2 --driver=docker
```

> `--memory=4096` з умови не пройшов: `MK_USAGE: Docker Desktop має лише 3916МБ памʼяті, але ви вказали 4096МБ`. Для демо з двома Pod-ами 3 ГБ достатньо.

```
$ minikube profile list
│ PROFILE  │ DRIVER │  RUNTIME   │      IP      │ VERSION │ STATUS │ NODES │ ACTIVE PROFILE │ ACTIVE KUBECONTEXT │
│ minikube │ docker │ containerd │ 192.168.49.2 │ v1.37.0 │ OK     │ 1     │ *              │ *                  │

$ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default

$ kubectl config current-context
minikube

$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   25s   v1.37.0
```

Контекст — **`minikube`**, нода `minikube` у статусі **Ready** (одразу після старту кілька секунд була `NotReady`, поки піднімалася мережа кластера).

### 4. Namespace, ConfigMap, Secret, Deployment, Service

Образ зібрано всередині Minikube під тим тегом, що в `k8s/deployment.yaml` (`imagePullPolicy: IfNotPresent` → kubelet бере локальний образ, а не з Docker Hub):

```bash
minikube image build -t maxnavrotsky/k8s-demo-app:1.0.0 .

kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

```
$ kubectl get configmap -n k8s-demo
NAME               DATA   AGE
k8s-demo-config    4      7s
kube-root-ca.crt   1      7s

$ kubectl get secret -n k8s-demo
NAME              TYPE     DATA   AGE
k8s-demo-secret   Opaque   1      7s

$ kubectl get deployment,rs,pods,svc -n k8s-demo
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           7s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-5c45559886   2         2         2       7s

NAME                                READY   STATUS    RESTARTS   AGE   IP
pod/k8s-demo-api-5c45559886-527z8   1/1     Running   0          7s    10.244.0.3
pod/k8s-demo-api-5c45559886-klc98   1/1     Running   0          7s    10.244.0.4

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.99.44.112   <none>        80/TCP    7s
```

Обидва Pod-и `Running` і `READY 1/1`.

### 5. Доступ через Service і port-forward

```bash
kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80
```

```
$ curl http://localhost:8080/api/status
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T13:06:36.290Z"}
$ curl http://localhost:8080/api/items
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]

$ kubectl get endpoints k8s-demo-api -n k8s-demo
NAME           ENDPOINTS                         AGE
k8s-demo-api   10.244.0.3:8080,10.244.0.4:8080   15s

$ kubectl logs -n k8s-demo -l app=k8s-demo-api --prefix --tail=3
[pod/k8s-demo-api-5c45559886-527z8/k8s-demo-api] [2026-11-24T13:06:36.291Z] GET /api/status 200 1ms
[pod/k8s-demo-api-5c45559886-527z8/k8s-demo-api] [2026-11-24T13:06:36.306Z] GET /api/items 200 2ms
```

Шлях запиту: `curl localhost:8080` → kubectl (тунель через API-сервер і kubelet) → **Service `k8s-demo-api`** (порт 80, selector `app=k8s-demo-api`) → **Pod** на `targetPort` 8080. Логи підтверджують, що відповідь дав Pod `…527z8` у кластері. Особливість: `port-forward svc/…` не балансує — kubectl один раз обирає Pod за selector-ом Service і всі з'єднання йдуть у нього.

---

## 75 балів

### 1. ConfigMap: стартове значення `APP_MODE=dev`

У fork-і `k8s/configmap.yaml` спершу мав `APP_MODE: "prod"`; для демонстрації переходу dev → prod стартове значення змінено на `"dev"`.

```
$ kubectl get configmap k8s-demo-config -n k8s-demo -o yaml
data:
  APP_MODE: dev
  ERROR_MODE: "off"
  READINESS_DELAY_MS: "5000"
  SLOW_MS: "1500"

$ kubectl rollout restart deployment k8s-demo-api -n k8s-demo
deployment.apps/k8s-demo-api restarted

$ kubectl get pods -n k8s-demo
NAME                            READY   STATUS    RESTARTS   AGE
k8s-demo-api-6cf477594f-br4dm   1/1     Running   0          37s
k8s-demo-api-6cf477594f-mgqth   1/1     Running   0          44s

$ kubectl exec -n k8s-demo k8s-demo-api-6cf477594f-br4dm -- env | grep APP_MODE
APP_MODE=dev

$ curl http://localhost:8080/api/status        # port-forward на pod/k8s-demo-api-6cf477594f-br4dm
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T13:07:30.547Z"}
```

### 2. Зміна ConfigMap без рестарту Pod-ів

`k8s/configmap.yaml`: `APP_MODE: "dev"` → `APP_MODE: "prod"`.

```
$ kubectl apply -f k8s/configmap.yaml
configmap/k8s-demo-config configured

$ kubectl get configmap k8s-demo-config -n k8s-demo -o yaml
data:
  APP_MODE: prod
  ...

$ kubectl exec -n k8s-demo k8s-demo-api-6cf477594f-br4dm -- env | grep APP_MODE   # той самий Pod
APP_MODE=dev

$ curl http://localhost:8080/api/status
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T13:07:51.382Z"}
```

**Чому в Pod-і досі `dev`:** Deployment підключає ConfigMap через `envFrom`. Змінні оточення kubelet підставляє **один раз — під час створення контейнера**, перед стартом процесу; далі вони живуть у пам'яті процесу Node.js (`process.env`) і ззовні не змінюються. `kubectl apply` оновив лише об'єкт ConfigMap в API-сервері, а запущені контейнери його не перечитують. (Автоматично оновлюються лише ConfigMap-и, змонтовані як **том**, — і то з затримкою до ~1 хв, а застосунок має сам перечитати файл.)

### 3. Rollout restart і нова конфігурація

```
$ kubectl rollout restart deployment k8s-demo-api -n k8s-demo
deployment.apps/k8s-demo-api restarted

$ kubectl get pods -n k8s-demo
NAME                            READY   STATUS    RESTARTS   AGE
k8s-demo-api-7d6fb54985-8cbqk   1/1     Running   0          46s
k8s-demo-api-7d6fb54985-k4nm7   1/1     Running   0          52s

$ kubectl exec -n k8s-demo k8s-demo-api-7d6fb54985-8cbqk -- env | grep APP_MODE
APP_MODE=prod

$ curl http://localhost:8080/api/status        # port-forward svc/k8s-demo-api
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-24T13:08:42.768Z"}
```

**Роль `rollout restart`:** команда додає в шаблон Pod-а анотацію `kubectl.kubernetes.io/restartedAt`. Шаблон змінився → Deployment створює новий ReplicaSet (`…6cf477594f` → `…7d6fb54985`) і за стратегією RollingUpdate поступово замінює Pod-и: новий Pod стає Ready → старий зупиняється. Нові контейнери стартують уже з поточним значенням ConfigMap. Сервіс при цьому не простоює.

**Рекомендований підхід до оновлення конфігурації з ConfigMap:**
1. Міняти ConfigMap у git (маніфест), не через `kubectl edit`.
2. `kubectl apply -f k8s/configmap.yaml`, потім `kubectl rollout restart deployment/<ім'я>` і `kubectl rollout status` для контролю.
3. Краще — робити рестарт автоматичним: додати в шаблон Pod-а анотацію з хешем ConfigMap (так роблять Helm/Kustomize, `configMapGenerator` дає ConfigMap з суфіксом-хешем) або використати незмінні (immutable) ConfigMap з новим ім'ям на кожну версію. Тоді зміна конфігурації = зміна шаблону = rolling update, і її видно в `kubectl rollout history` та можна відкотити `rollout undo`.

#### Знахідка: старий Pod на мить у статусі `Error`

Під час rollout один зі старих Pod-ів показав `0/1 Error`:

```
NAME                            READY   STATUS    RESTARTS   AGE
k8s-demo-api-6cf477594f-mgqth   0/1     Error     0          116s
k8s-demo-api-7d6fb54985-8cbqk   1/1     Running   0          36s
```

Причина: Node.js у контейнері — процес з PID 1, а `index.js` не обробляє SIGTERM. Для PID 1 ядро не застосовує стандартну дію сигналу, тож SIGTERM ігнорується, kubelet чекає `terminationGracePeriodSeconds` (30 с) і вбиває процес SIGKILL → код виходу 137 → статус `Error`. Перевірено локально:

```
$ docker run -d --name sigterm-test -e SECRET_TOKEN=x k8s-demo-app:1.0.0
$ docker stop sigterm-test
$ docker inspect sigterm-test --format 'ExitCode={{.State.ExitCode}}'
ExitCode=137
```

На трафік це не впливає (Pod уже прибраний з endpoints Service), але кожен rollout затягується на 30 с. Виправлення: обробити SIGTERM у застосунку (`process.on('SIGTERM', () => server.close(() => process.exit(0)))`) або запускати через init (`docker run --init` / `tini`).

### 4. Liveness і readiness

```
$ kubectl describe pod k8s-demo-api-7d6fb54985-8cbqk -n k8s-demo
    Liveness:   http-get http://:8080/healthz delay=5s timeout=1s period=10s #success=1 #failure=3
    Readiness:  http-get http://:8080/ready delay=3s timeout=1s period=5s #success=1 #failure=3
    Environment Variables from:
      k8s-demo-config  ConfigMap  Optional: false
      k8s-demo-secret  Secret     Optional: false
```

| Перевірка | Шлях | initialDelaySeconds | periodSeconds | failureThreshold |
|---|---|---|---|---|
| Liveness | `/healthz` | 5 | 10 | 3 (за замовчуванням) |
| Readiness | `/ready` | 3 | 5 | 3 (за замовчуванням) |

`failureThreshold` у `deployment.yaml` не задано — `describe` показує значення за замовчуванням `#failure=3`.

**Різниця:**
- **Liveness** відповідає на питання «процес живий чи завис?». Якщо `/healthz` не відповідає 3 рази поспіль (30 с), kubelet **перезапускає контейнер** (росте `RESTARTS`).
- **Readiness** — «чи готовий Pod приймати трафік?». Якщо `/ready` повертає не 2xx, Pod **виключається з endpoints Service**, але не перезапускається. Тут `/ready` віддає 503 перші 5 с після старту (`READINESS_DELAY_MS`), тому новий Pod спершу `0/1` і трафік на нього не йде, доки він не «прогріється» (видно в розділі 90 → самовідновлення).

---

## 90 балів

### 1. Ingress і `k8s-demo.local`

```bash
minikube addons enable ingress
minikube image build -t maxnavrotsky/k8s-demo-app:1.0.0 .   # уже виконано на кроці 60.4
kubectl apply -f k8s/ingress.yaml -n k8s-demo
echo "127.0.0.1 k8s-demo.local" | sudo tee -a /etc/hosts
sudo minikube tunnel        # окремий термінал, тримати відкритим
```

```
$ kubectl get ingress -n k8s-demo
NAME               CLASS   HOSTS            ADDRESS        PORTS   AGE
k8s-demo-ingress   nginx   k8s-demo.local   192.168.49.2   80      2m53s

$ kubectl describe ingress k8s-demo-ingress -n k8s-demo
Name:             k8s-demo-ingress
Namespace:        k8s-demo
Ingress Class:    nginx
Rules:
  Host            Path  Backends
  ----            ----  --------
  k8s-demo.local
                  /   k8s-demo-api:80 (10.244.0.7:8080,10.244.0.8:8080)
Events:
  Normal  Sync    21s   nginx-ingress-controller  Scheduled for sync
```

Доступ за доменом (запущено `sudo minikube tunnel`, у `/etc/hosts` — `127.0.0.1 k8s-demo.local`):

```
$ curl http://k8s-demo.local/api/status
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-24T13:15:24.318Z"}      [200]
$ curl http://k8s-demo.local/api/items
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]      [200]
$ curl 'http://k8s-demo.local/api/items?filter=b'
[{"id":2,"name":"Item B"}]      [200]

$ kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=3
"GET /api/status HTTP/1.1" 200 89 ... [k8s-demo-k8s-demo-api-80] [] 10.244.0.12:8080 89 0.004 200
"GET /api/items HTTP/1.1" 200 76 ... [k8s-demo-k8s-demo-api-80] [] 10.244.0.12:8080 76 0.005 200
"GET /api/items?filter=b HTTP/1.1" 200 26 ... [k8s-demo-k8s-demo-api-80] [] 10.244.0.7:8080 26 0.005 200
```

Останній запит обслужив інший Pod (`10.244.0.7`), тобто через Ingress трафік балансується між репліками — на відміну від `port-forward`.

Маршрутизацію Ingress додатково перевірено без `/etc/hosts`: port-forward на ingress-контролер і запит із заголовком `Host`:

```
$ kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8088:80
$ curl -H 'Host: k8s-demo.local' http://localhost:8088/api/status
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-24T13:12:47.925Z"}      [200]
$ curl -H 'Host: k8s-demo.local' http://localhost:8088/api/items
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]      [200]
$ curl -H 'Host: other.local' http://localhost:8088/api/status
[404]

$ kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=2
"GET /api/status HTTP/1.1" 200 89 ... [k8s-demo-k8s-demo-api-80] [] 10.244.0.12:8080 89 0.005 200
"GET /api/items HTTP/1.1" 200 76 ... [k8s-demo-k8s-demo-api-80] [] 10.244.0.12:8080 76 0.002 200
```

Host у `k8s/ingress.yaml`: **`k8s-demo.local`**, path `/` (Prefix) → Service `k8s-demo-api:80`. Запит з іншим Host отримує 404 — правило спрацьовує саме за Host.

**Шлях запиту `http://k8s-demo.local/api/status`:**
1. `/etc/hosts` перетворює `k8s-demo.local` на `127.0.0.1`.
2. `minikube tunnel` прокидає `127.0.0.1:80` з macOS у кластер (з драйвером docker нода Minikube недоступна з хоста напряму).
3. **Ingress-контролер** `ingress-nginx` (namespace `ingress-nginx`) приймає запит і за правилом Ingress `k8s-demo-ingress` (Host `k8s-demo.local`, path `/`) обирає бекенд.
4. **Service** `k8s-demo-api:80` — nginx бере з нього список endpoints (у лозі upstream `k8s-demo-k8s-demo-api-80`).
5. **Pod** `10.244.0.12:8080` — контейнер `k8s-demo-api` відповідає JSON-ом.

### 2. Debug-флоу

Помилок з Ingress не було (контролер прийняв конфігурацію: `successfully validated configuration, accepting`, `Backend successfully reloaded`). Проблеми, що траплялися по ходу лаби, і як їх знайдено:

| Проблема | Як діагностовано |
|---|---|
| `minikube start --memory=4096` падає | повідомлення `MK_USAGE` у виводі `minikube start` → зменшено до 3072 МБ |
| Старий Pod `Error` під час rollout | `kubectl get pods`, `kubectl get pod … -o jsonpath='{.status.containerStatuses[0].state.terminated}'`, локальний `docker stop` → exit 137 (див. розділ 75) |
| `curl k8s-demo.local` → таймаут, поки не запущено тунель | `kubectl get ingress` (ADDRESS 192.168.49.2 — IP ноди всередині Docker, з macOS недосяжний) → потрібен `minikube tunnel` |

Базові команди для діагностики доступу до сервісу:

| Команда | Що показує |
|---|---|
| `kubectl get pods -n k8s-demo -o wide` | чи Pod-и `Running` і `READY 1/1`, кількість рестартів, IP і ноду. `0/1` — не пройшла readiness, `CrashLoopBackOff` — процес падає, `ImagePullBackOff` — не знайдено образ |
| `kubectl describe pod <pod> -n k8s-demo` | образ, probes, env з ConfigMap/Secret і **Events**: невдалі probes (`Unhealthy`), помилки pull образу, OOMKilled, причини рестартів |
| `kubectl logs <pod> -n k8s-demo` (`--previous` для впалого контейнера) | stdout застосунку: стартові помилки, стек-трейси, лог кожного запиту |
| `kubectl describe svc k8s-demo-api -n k8s-demo` / `kubectl get endpointslices -n k8s-demo` | selector і **Endpoints**. Порожні endpoints = selector не збігається з labels Pod-ів або жоден Pod не Ready |
| `kubectl describe ingress k8s-demo-ingress -n k8s-demo` | Host, path і бекенд з IP Pod-ів; Events синхронізації. `<error: endpoints not found>` — неправильне ім'я/порт Service |
| `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` | access-лог nginx: статус, upstream (`namespace-service-port`) і IP Pod-а, що відповів; помилки валідації Ingress |
| `kubectl get events -n k8s-demo --sort-by=.metadata.creationTimestamp` | хронологія подій namespace: scheduling, створення/видалення Pod-ів ReplicaSet-ом, Killing, probes |

Фрагменти виводу:

```
$ kubectl describe svc k8s-demo-api -n k8s-demo
Selector:                 app=k8s-demo-api
Type:                     ClusterIP
IP:                       10.99.44.112
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
Endpoints:                10.244.0.7:8080,10.244.0.8:8080

$ kubectl get endpointslices -n k8s-demo
NAME                 ADDRESSTYPE   PORTS   ENDPOINTS               AGE
k8s-demo-api-6kxgf   IPv4          8080    10.244.0.7,10.244.0.8   3m43s

$ kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=6
"successfully validated configuration, accepting" ingress="k8s-demo/k8s-demo-ingress"
"Found valid IngressClass" ingress="k8s-demo/k8s-demo-ingress" ingressclass="nginx"
"Configuration changes detected, backend reload required"
"Backend successfully reloaded"

$ kubectl get events -n k8s-demo --sort-by=.metadata.creationTimestamp | tail
Normal   ScalingReplicaSet   deployment/k8s-demo-api              Scaled up replica set k8s-demo-api-7d6fb54985 from 1 to 2
Normal   Killing             pod/k8s-demo-api-6cf477594f-mgqth    Stopping container k8s-demo-api
Normal   ScalingReplicaSet   deployment/k8s-demo-api              Scaled down replica set k8s-demo-api-6cf477594f from 1 to 0
Normal   Sync                ingress/k8s-demo-ingress             Scheduled for sync
```

### 3. Самовідновлення Pod-ів

```
$ kubectl get pods -n k8s-demo
NAME                            READY   STATUS    RESTARTS   AGE
k8s-demo-api-7d6fb54985-8cbqk   1/1     Running   0          2m5s
k8s-demo-api-7d6fb54985-k4nm7   1/1     Running   0          2m11s

$ kubectl delete pod k8s-demo-api-7d6fb54985-8cbqk -n k8s-demo      # 15:10:08
pod "k8s-demo-api-7d6fb54985-8cbqk" deleted from k8s-demo namespace

$ kubectl get pods -n k8s-demo                                       # через 1 с
NAME                            READY   STATUS        RESTARTS   AGE
k8s-demo-api-7d6fb54985-8cbqk   1/1     Terminating   0          2m7s
k8s-demo-api-7d6fb54985-fqw2d   0/1     Running       0          2s
k8s-demo-api-7d6fb54985-k4nm7   1/1     Running       0          2m13s

$ kubectl get pods -n k8s-demo                                       # через ~12 с
NAME                            READY   STATUS        RESTARTS   AGE
k8s-demo-api-7d6fb54985-8cbqk   1/1     Terminating   0          2m19s
k8s-demo-api-7d6fb54985-fqw2d   1/1     Running       0          14s
k8s-demo-api-7d6fb54985-k4nm7   1/1     Running       0          2m25s

$ kubectl get rs -n k8s-demo -l app=k8s-demo-api
NAME                      DESIRED   CURRENT   READY   AGE
k8s-demo-api-7d6fb54985   2         2         2       2m25s

$ kubectl get events -n k8s-demo --sort-by=.metadata.creationTimestamp | grep -E "SuccessfulCreate|Killing" | tail -2
14s   Normal   Killing            pod/k8s-demo-api-7d6fb54985-8cbqk    Stopping container k8s-demo-api
13s   Normal   SuccessfulCreate   replicaset/k8s-demo-api-7d6fb54985   Created pod: k8s-demo-api-7d6fb54985-fqw2d
```

**Що відбулося:** щойно Pod `…8cbqk` перейшов у `Terminating`, ReplicaSet уже за ~1 с створив замість нього `…fqw2d`. Новий Pod спершу `0/1` (readiness `/ready` віддає 503 перші 5 с), тож Service не слав на нього трафік; через ~12 с він став `1/1`. Кількість робочих Pod-ів повернулася до 2. Старий Pod ще ~30 с висить у `Terminating` через ігнорування SIGTERM (див. розділ 75).

**Чому Pod створився сам:** Deployment не керує Pod-ами напряму — він тримає ReplicaSet поточної ревізії з `replicas: 2`. Контролер ReplicaSet у циклі звіряє бажаний стан (2 Pod-и з label `app=k8s-demo-api, pod-template-hash=7d6fb54985`) з фактичним. Pod у `Terminating` уже не рахується, фактично лишився 1 → контролер створює новий Pod за шаблоном. Pod-и — «одноразові»: новий має інше ім'я та IP, а стабільну адресу дає Service.

---

## Контрольні запитання

**a. ConfigMap vs Secret.** Обидва — пари ключ-значення, які підключаються в Pod як env (`envFrom`/`valueFrom`) або як файли в томі. ConfigMap — для неконфіденційних налаштувань (`APP_MODE`, `SLOW_MS`), зберігається відкритим текстом. Secret — для паролів, токенів, ключів (`SECRET_TOKEN`): значення в маніфесті в base64 (це **кодування, не шифрування**: `echo bG9jYWwtdG9rZW4= | base64 -d` → `local-token`), але API-сервер обробляє їх окремо — доступ можна обмежити через RBAC, у etcd можна ввімкнути шифрування, `kubectl describe` не показує значення, а в томі Secret монтується в tmpfs (пам'ять, не диск). Тому реальні секрети не комітять у git, як у демо, а створюють з CI або через Sealed Secrets / External Secrets.

**b. Liveness vs readiness.** Liveness перевіряє, чи процес живий; провал → kubelet **перезапускає контейнер**. Readiness перевіряє, чи Pod готовий обслуговувати запити; провал → Pod **прибирається з endpoints Service** без рестарту. Тут liveness — `/healthz` (завжди `ok`), readiness — `/ready` (503 перші 5 с після старту). Помилкою було б зробити liveness на повільну залежність (БД): її збій спричинив би безкінечні рестарти здорових контейнерів.

**c. Шлях запиту:**
- **`kubectl port-forward svc/k8s-demo-api 8080:80`:** `localhost:8080` → процес kubectl → API-сервер → kubelet на ноді → **Pod** на порт 8080. Service тут використовується лише щоб знайти Pod за selector-ом і перекласти порт 80 → targetPort 8080; ClusterIP і kube-proxy не задіяні, балансування немає (усі з'єднання в один Pod).
- **`http://k8s-demo.local`:** `/etc/hosts` → `127.0.0.1` → `minikube tunnel` → **ingress-nginx controller** → правило **Ingress** `k8s-demo-ingress` (Host `k8s-demo.local`, path `/`) → **Service** `k8s-demo-api:80` (endpoints) → **Pod** `:8080`. Тут трафік балансується між усіма Ready Pod-ами.

**d. Оновлення конфігурації з env-змінних ConfigMap.** Env читаються лише під час старту контейнера, тому після `kubectl apply -f k8s/configmap.yaml` потрібен `kubectl rollout restart deployment/k8s-demo-api -n k8s-demo` (і `kubectl rollout status` для контролю) — rolling update створить нові Pod-и з новими значеннями без простою. Надійніше автоматизувати: хеш ConfigMap в анотації шаблону Pod-а або ConfigMap з новим ім'ям на кожну версію (Kustomize `configMapGenerator`, Helm `checksum/config`), щоб зміна конфігурації сама запускала rollout і була в історії ревізій.

**e. (рівень 100)** — див. розділ 100 (Розробник 2).

---

## 100 балів: StudySwap у Kubernetes

_Розділ заповнює Розробник 2._
