# Kubernetes: вопросы на интервью

Senior-уровень. Предполагается глубокое знание — сразу к внутренностям и failure-сценариям.

---

## Архитектура и control plane

### В: Опиши полный жизненный цикл пода — от `kubectl apply` до работающего контейнера.

**Это самый частый K8s-вопрос на senior-интервью. Нужно знать каждый компонент.**

```
kubectl apply -f pod.yaml
  │
  ▼
[1] kubectl → локальная валидация манифеста (схема), затем HTTP POST на kube-apiserver
  │
  ▼
[2] kube-apiserver
    - аутентификация (mTLS cert / token / OIDC)
    - авторизация (RBAC: может ли этот пользователь создавать поды в этом namespace?)
    - admission controllers (сначала MutatingAdmissionWebhook, затем ValidatingAdmissionWebhook)
      → mutating: может инжектировать sidecar, устанавливать дефолтные лимиты, добавлять env vars
      → validating: может отклонить (например, образ не подписан, нарушение PodSecurityAdmission)
    - запись в etcd (объект сохранён, status: Pending)
    - возвращает 201 Created в kubectl
  │
  ▼
[3] kube-scheduler следит за подами с .spec.nodeName == ""
    - Фаза Filter: убрать ноды, которые не могут запустить под
      → resource requests (CPU/memory), node selectors, taints/tolerations, pod affinity,
         конфликты портов, node affinity PV
    - Фаза Score: ранжировать оставшиеся ноды
      → LeastAllocated, NodeAffinity score, ImageLocality (нода уже имеет образ)
    - Записывает .spec.nodeName = "node-x" обратно в apiserver → etcd
  │
  ▼
[4] kubelet на node-x следит за подами, привязанными к его ноде
    - вызывает container runtime через CRI (containerd или CRI-O)
    - скачивает образ, если нет в кеше (проверяет image pull policy)
  │
  ▼
[5] containerd / CRI-O
    - скачивает образ (если нужно), проверяет digest
    - настраивает pod sandbox: вызывает CNI плагин → создаёт network namespace, veth pair,
      назначает IP из IPAM, настраивает маршруты
    - создаёт init containers (если есть), ждёт завершения
    - создаёт app containers
  │
  ▼
[6] kubelet запускает пробы
    - startupProbe (если задана): ждёт, пока приложение инициализируется
    - livenessProbe: перезапускает контейнер при неудаче
    - readinessProbe: добавляет под в endpoints сервиса только после успеха
  │
  ▼
[7] kube-proxy (или eBPF) следит за endpoints — добавляет iptables/IPVS правила
    для маршрутизации трафика сервиса (ClusterIP) на IP этого пода

Статус пода → Running, condition Ready=True
```

**Что проверяют интервьюеры:** упомянул ли admission controllers? Знаешь ли разницу filter/score в scheduling? Упомянул ли CNI для настройки сети? Понимаешь ли разницу readiness vs liveness по времени?

---

### В: Как kube-scheduler принимает решение о размещении?

**Сильный ответ:**

Две фазы: **Filter** (исключить) и **Score** (ранжировать).

**Плагины Filter (любой провал = нода исключена):**
- `NodeResourcesFit`: у ноды достаточно CPU/памяти для requests пода
- `NodeSelector`: `nodeSelector` пода совпадает с лейблами ноды
- `TaintToleration`: под допускает все taint'ы ноды
- `PodTopologySpread`: не нарушает `topologySpreadConstraints`
- `VolumeBinding`: PVC может быть привязан на этой ноде (важно для local PV)
- `NodeAffinity`: обязательные правила node affinity
- `InterPodAffinity`: обязательные правила pod affinity/anti-affinity

**Плагины Score (0-100, взвешенная сумма):**
- `LeastAllocated`: предпочитать ноды с наибольшим количеством свободных ресурсов
- `NodeAffinity`: предпочтительные правила affinity
- `ImageLocality`: предпочитать ноды, у которых уже есть образ (избегает задержки pull)
- `PodTopologySpread`: равномерное распределение по зонам

**Кастомные schedulers:** можно запускать несколько schedulers. Поды ссылаются на scheduler по имени в `spec.schedulerName`. Используется для GPU-нагрузок, специализированной логики размещения.

**Что ломается при масштабировании:**
- При тысячах нод scheduler семплирует подмножество (настраивается через `percentageOfNodesToScore`, по умолчанию 50% или минимум 100)
- Scheduler — единственный активный инстанс (leader election) — bottleneck при высокой частоте создания подов

---

### В: etcd показывает высокую латентность. Какой операционный impact и как восстановиться?

**Сильный ответ:**

etcd — мозг Kubernetes. Каждая операция control plane читает/пишет в etcd. Высокая латентность каскадирует:

**Цепочка impact:**
1. Запросы kube-apiserver к etcd таймаутят → apiserver возвращает 503
2. kubectl-команды зависают или падают
3. kube-scheduler не может записать привязку пода → поды остаются в Pending
4. kubelet не может обновить статус пода → поды выглядят застрявшими
5. HPA, controllers не могут reconcile → желаемое состояние не применяется

**Диагностика:**
```bash
# Здоровье etcd
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster -w table   # лидер, размер DB, raft lag

# Метрики для проверки
etcd_disk_wal_fsync_duration_seconds         # > 10ms — повод беспокоиться
etcd_disk_backend_commit_duration_seconds    # > 25ms — плохо
etcd_server_leader_changes_total             # частая смена лидера = нестабильность
etcd_mvcc_db_total_size_in_bytes             # > 8GB → нужна дефрагментация

# Со стороны apiserver
apiserver_request_duration_seconds{verb="LIST"}  # должно быть < 1s
```

**Шаги восстановления:**
1. **Немедленно**: если один member отстаёт — проверить дисковый I/O (`iostat`) — etcd крайне чувствителен к задержкам, fsync должен быть < 10ms → SSD, изолировать от шумных соседей
2. **Дефрагментация при большой DB**: `etcdctl defrag --cluster` (кратко блокирует каждый member)
3. **Компактизация**: `etcdctl compact $(etcdctl endpoint status --write-out="json" | jq '.[0].Status.header.revision')`
4. **Замена member при повреждении**: удалить member, восстановить из snapshot, добавить обратно
5. **Никогда** не запускать etcd на shared-дисках или рядом с шумными I/O соседями

**Snapshot/restore:**
```bash
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db
etcdctl snapshot restore /backup/etcd-latest.db --data-dir /var/lib/etcd-new
```

---

## Сеть

### В: Как работает сеть в Kubernetes? Опиши путь пакета от пода A до пода B на разных нодах.

**Полный путь:**

```
Pod A (node1, IP 10.0.1.5) → Pod B (node2, IP 10.0.2.7)

[Контейнер Pod A]
  └─ пишет в eth0 (veth pair внутри network namespace)
       │
[Ядро node1]
  └─ veth pair: пакет появляется на мосту cbr0/cni0 (или маршрутизируется напрямую с Cilium)
  └─ таблица маршрутизации: 10.0.2.0/24 через 192.168.1.2 (IP node2)
  └─ инкапсуляция (зависит от CNI):
       - Flannel VXLAN: оборачивает в UDP, отправляет на node2:8472
       - Calico BGP: без инкапсуляции, маршрутизация на L3
       - Cilium eBPF: минует большую часть этого, программирует XDP/TC hooks
       │
[Ядро node2]
  └─ декапсуляция (если VXLAN)
  └─ маршрутизация на 10.0.2.7 → veth pair → network namespace Pod B
```

**Путь через Service (ClusterIP):**
```
Pod A → ClusterIP (например, 10.96.0.10)
  └─ kube-proxy запрограммировал правила DNAT в iptables:
     PREROUTING: -d 10.96.0.10 -j DNAT --to-destination <pod-IP>:<port>
     (случайно выбирает один из IP endpoints)
  └─ пакет идёт на выбранный IP пода, дальше — как описано выше
```

**Почему iptables ломается при масштабировании:**
- Каждый сервис = O(n_endpoints) iptables-правил
- Оценка правила — O(n_правил): последовательный скан
- При ~10 000 сервисов: `kube-proxy` синхронизируется минутами, конкуренция за блокировку iptables
- **Решение**: переключиться на режим IPVS (`--proxy-mode=ipvs`) — поиск за O(1) через хэш-таблицу

**Cilium eBPF**: полностью обходит iptables, программирует XDP и TC hooks, прямая pod-to-pod связь без bridge. Позволяет применять network policy на L7 (HTTP path, gRPC method).

---

## Отладка

### В: Сервис проходит health checks, но возвращает 503 для ~3% запросов. Как диагностировать?

**Сначала mitigation:** проверить, растёт ли error rate. Если да — масштабировать или временно убрать сервис из load balancer, пока разбираемся.

**Путь диагностики:**

```bash
# 1. Один под виноват или рандомно?
kubectl get pods -o wide          # записать IP подов и ноды
# Сопоставить 503 с конкретными IP подов из access logs

# 2. Проверить readiness подов
kubectl describe pod <pod>        # сбои readiness probe?
kubectl get endpoints <service>   # все ли поды зарегистрированы?

# 3. 3% намекает на специфические условия — проверить:
# - Connection draining: один под завершается (timing preStop hook)?
kubectl get events --sort-by='.lastTimestamp'

# 4. Согласованность kube-proxy / iptables
iptables-save | grep <service-clusterip>   # правильные DNAT-правила?
ipvsadm -L -n                              # если режим IPVS

# 5. NetworkPolicy блокирует?
kubectl get networkpolicy -n <namespace>

# 6. Resource pressure вызывает медленные ответы, которые не проходят health check?
kubectl top pods
kubectl describe node <node>      # Conditions, allocatable vs requested

# 7. Уровень приложения
kubectl logs <pod> --previous     # недавно падал?
# Проверить логи приложения на паттерн ошибок вокруг времени 503
```

**Частые причины 3% ошибок:**
- Rolling update пода: завершающийся под ещё в списке endpoints → добавить `preStop: sleep 5`
- Слишком агрессивная readiness probe → под кратко убирается и добавляется обратно
- Переполнена таблица ConnTrack на одной ноде: `sysctl net.netfilter.nf_conntrack_count` vs `max`
- Disk pressure на одной ноде → kubelet evicting → гонка с SIGTERM

---

### В: В чём разница между Pending, CrashLoopBackOff и OOMKilled?

**Pending:**
Под запланирован или застрял до scheduling. Первые три команды:
```bash
kubectl describe pod <pod>   # смотреть секцию Events
# Частые причины:
# - "0/3 nodes available: 3 Insufficient memory" → requests слишком большие или ноды заняты
# - "didn't match node selector" → несовпадение nodeSelector/affinity
# - "had taint that pod didn't tolerate" → добавить toleration
# - "Unschedulable: pvc not bound" → проблема StorageClass
```

**CrashLoopBackOff:**
Контейнер запускается, завершается (ненулевой код), kubelet перезапускает с экспоненциальным backoff (10s → 20s → 40s... максимум 5 мин).
```bash
kubectl logs <pod>             # текущие логи
kubectl logs <pod> --previous  # логи ПОСЛЕДНЕГО краша — это то, что нужно
kubectl describe pod <pod>     # Exit Code в Last State
# Exit Code 1:   ошибка приложения
# Exit Code 137: SIGKILL (OOM или ручное убийство)
# Exit Code 139: SIGSEGV (segfault)
# Exit Code 143: SIGTERM (graceful shutdown, завершился с ненулевым кодом)
```

**OOMKilled:**
Контейнер превысил лимит памяти → ядро убило его (Exit Code 137).
```bash
kubectl describe pod <pod>    # "OOMKilled" в Last State Reason
kubectl top pod <pod>         # текущее потребление памяти
# Проверить: лимит слишком мал или утечка памяти?
# Смотреть тренд памяти: kubectl top pod -w
# Исправить: увеличить лимиты или исправить утечку
# Важно: OOMKilled от cgroup limit ≠ системный OOM killer
```

---

## Безопасность

### В: Объясни как работает RBAC в Kubernetes.

**Сильный ответ:**

RBAC контролирует, какие **глаголы** (get, list, create, delete, patch, watch) можно применять к **ресурсам** (pods, secrets, deployments) в каком **scope** (namespace или кластер).

**Компоненты:**
```
Role              → глаголы + ресурсы, namespace-scoped
ClusterRole       → то же, но для всего кластера (или для non-namespaced ресурсов, например nodes)
RoleBinding       → привязывает Role/ClusterRole к subjects в namespace
ClusterRoleBinding → привязывает ClusterRole к subjects в масштабе кластера
```

**Subjects:** `User`, `Group`, `ServiceAccount`

**Пример:**
```yaml
# ServiceAccount для CI-системы, которая может деплоить
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]
---
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: ci
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Аудит избыточных прав:**
```bash
# Что может делать service account?
kubectl auth can-i --list --as=system:serviceaccount:default:myapp

# Кто может получить доступ к secrets в namespace?
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name | test("secret|admin")) | .subjects'

# Инструменты: rbac-audit, rakkess, kubectl-who-can
kubectl-who-can get secrets -n production
```

**Частая ошибка:** выдать `ClusterAdmin` service account «временно» и забыть убрать. Проверяй регулярно.

---

## Масштабирование и надёжность

### В: Как работает HPA изнутри? Failure modes при большом масштабе?

**Сильный ответ:**

HPA controller запускается каждые 15 секунд (настраивается). Алгоритм:
```
desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))
```

**Источники метрик:**
1. `metrics.k8s.io` (CPU/memory) — предоставляет metrics-server, который опрашивает kubelet каждые 60s
2. `custom.metrics.k8s.io` — адаптер (Prometheus Adapter) читает кастомные метрики из Prometheus
3. `external.metrics.k8s.io` — внешние источники (Datadog, Stackdriver)

**Failure modes:**

1. **metrics-server недоступен** → HPA не может масштабировать → события показывают `unable to get metrics`
2. **Лаг метрик**: всплеск CPU на 30 секунд может быть пропущен (интервал scrape 60s, интервал HPA 15s, но использует последний семпл)
3. **Scale-down thrashing**: дефолтный `--horizontal-pod-autoscaler-downscale-stabilization=5m` предотвращает быстрый scale-down, но вызывает избыточное выделение ресурсов
4. **Конфликт с maxUnavailable**: HPA хочет scale down, PDB блокирует eviction → застрял
5. **Кардинальность custom metrics**: если запрос Prometheus Adapter дорогой — таймауты адаптера каскадируют в то, что HPA не масштабирует

**При масштабировании (тысячи подов):**
- HPA делает `List` на подах для получения метрик — при большом масштабе эти LIST-запросы нагружают apiserver
- Рассмотреть KEDA (Kubernetes Event-Driven Autoscaling) — более сложные источники метрик и логика масштабирования

---

## Шпаргалка: отладка K8s

```bash
# Под не запускается?
kubectl get pod <pod> -o yaml      # полный spec + status
kubectl describe pod <pod>         # события (самое полезное)
kubectl logs <pod> -c <container>  # логи контейнера
kubectl logs <pod> --previous      # предыдущий контейнер (после краша)

# Проблемы с нодой?
kubectl describe node <node>       # conditions, allocatable, events
kubectl get events --field-selector reason=OOMKilling

# Сеть?
kubectl exec -it <pod> -- curl -v http://<service>:<port>
kubectl exec -it <pod> -- nslookup <service>.<namespace>.svc.cluster.local

# Resource pressure?
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Endpoints сервиса?
kubectl get endpoints <service>    # пусто = нет ready-подов, совпадающих с selector
```
