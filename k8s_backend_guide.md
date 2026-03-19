# Kubernetes для backend-разработчика: практический гайд

> Этот гайд — не про архитектуру etcd и не про настройку кластеров.
> Это про то, что ты будешь делать **каждый день** на работе.
> Написано разработчиком с 10+ годами опыта для тех, кто выходит на первую работу с Kubernetes.

---

## Введение: чего ожидать

Слушай. Ты прошёл уроки по Kubernetes, знаешь что такое Pod, Deployment, Service. Это хорошо. Но между «знаю что такое Pod» и «на production упал сервис в 3 часа ночи, ты on-call дежурный» — пропасть.

Этот гайд про вторую часть. Я расскажу что ты будешь реально делать, какие грабли встретишь, и как не выглядеть растерянным, когда тимлид скажет «разберись, почему order-service не стартует».

Я не буду рассказывать как поднять кластер. Это не твоя работа. Кластер уже есть. Твоя работа — чтобы **твой сервис** в этом кластере работал.

---

## Часть 1. Твой первый день: ориентация

### Что ты увидишь

Придёшь на работу. Тебе дадут доступ к кластеру. Скорее всего, скинут файл `kubeconfig` или скажут выполнить что-то вроде:

```bash
# AWS EKS
aws eks update-kubeconfig --name production-cluster --region eu-west-1

# Google GKE
gcloud container clusters get-credentials production --zone europe-west1

# Или просто
cp ~/Downloads/kubeconfig ~/.kube/config
```

Первое что делаешь — проверяешь, что доступ работает:

```bash
kubectl get nodes
```

Увидишь что-то вроде:

```
NAME                          STATUS   ROLES    AGE    VERSION
ip-10-0-1-15.ec2.internal     Ready    <none>   45d    v1.35.2
ip-10-0-1-16.ec2.internal     Ready    <none>   45d    v1.35.2
ip-10-0-2-22.ec2.internal     Ready    <none>   45d    v1.35.2
ip-10-0-2-23.ec2.internal     Ready    <none>   12d    v1.35.2
```

Четыре узла. Это **чужая территория** — узлами занимается платформенная команда. Тебе сюда не надо.

### Namespaces — твоя комната в общежитии

В большой компании один кластер делят между командами. У каждой — свой namespace:

```bash
kubectl get namespaces
```

```
NAME              STATUS   AGE
default           Active   120d
kube-system       Active   120d
orders-team       Active   90d
payments-team     Active   90d
catalog-team      Active   85d
staging           Active   60d
```

Тебе скажут: «Ты в команде orders, твой namespace — `orders-team`». С этого момента **всегда** указывай namespace:

```bash
# Устанавливаешь namespace по умолчанию — чтобы не писать -n каждый раз
kubectl config set-context --current --namespace=orders-team
```

Это важно. Я видел как джуниор на второй неделе работы случайно удалил Deployment в чужом namespace, потому что забыл переключиться. Ничего страшного не произошло (Deployment пересоздали), но нервы потрепал всем. Настрой свой namespace сразу.

### Осмотрись

```bash
# Что вообще работает в твоём namespace
kubectl get all

# Более подробно
kubectl get deployments
kubectl get services
kubectl get pods
kubectl get configmaps
kubectl get secrets
kubectl get ingress
```

Не бойся этих команд — они **ничего не меняют**, только показывают. `get` — read-only. Смотри сколько хочешь.

---

## Часть 2. Повседневные задачи

### Задача №1: «Посмотри логи, разберись что случилось»

Это 50% твоего взаимодействия с Kubernetes. Что-то не работает — первым делом логи.

```bash
# Логи конкретного Pod'а
kubectl logs order-service-7d8f9b5c6-abc12

# Проблема: ты не помнишь имя Pod'а (оно и правда безумное).
# Решение: используй label selector
kubectl logs -l app=order-service

# А если Pod'ов несколько (replicas: 3)?
# Добавь --all-containers и --prefix (покажет от какого Pod'а строка)
kubectl logs -l app=order-service --all-containers --prefix

# Follow — смотреть логи в реальном времени (как tail -f)
kubectl logs -f -l app=order-service

# Логи за последний час (не все 10 гигабайт за неделю)
kubectl logs -l app=order-service --since=1h

# Логи предыдущего (упавшего) контейнера
# Это КРИТИЧЕСКИ ВАЖНО. Pod перезапустился — логи текущего контейнера пустые.
# А ошибка была в предыдущем. Вот как их достать:
kubectl logs order-service-7d8f9b5c6-abc12 --previous
```

**Реальная история**: приходит алерт — «order-service 5xx > 1%». Я смотрю логи текущего Pod'а — всё чисто. Потому что Pod перезапустился и ошибки были в **предыдущем** контейнере. 20 минут потерял, пока не вспомнил про `--previous`. Не повторяй.

### Задача №2: «Почему Pod не стартует?»

Рано или поздно ты увидишь это:

```bash
kubectl get pods
```

```
NAME                            READY   STATUS             RESTARTS   AGE
order-service-7d8f9b5c6-abc12   0/1     CrashLoopBackOff   5          3m
```

`CrashLoopBackOff` — Pod запускается, падает, Kubernetes перезапускает, снова падает, снова перезапускает... с нарастающей задержкой (10с, 20с, 40с...).

**Алгоритм разбора (запомни его наизусть):**

```bash
# Шаг 1: Describe — покажет СОБЫТИЯ (events)
kubectl describe pod order-service-7d8f9b5c6-abc12
```

Скролль до конца, секция `Events`:

```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  3m                 default-scheduler  Successfully assigned...
  Normal   Pulled     3m (x5 over 3m)   kubelet            Container image pulled
  Normal   Created    3m (x5 over 3m)   kubelet            Created container
  Normal   Started    3m (x5 over 3m)   kubelet            Started container
  Warning  BackOff    10s (x15 over 3m) kubelet            Back-off restarting failed container
```

Events говорят **что делал Kubernetes**. Но не говорят **почему приложение упало**. Для этого — логи.

```bash
# Шаг 2: Логи (текущего или предыдущего контейнера)
kubectl logs order-service-7d8f9b5c6-abc12
# или
kubectl logs order-service-7d8f9b5c6-abc12 --previous
```

И вот тут ты увидишь **настоящую причину**. В 90% случаев это одно из:

**Причина 1: Приложение не может подключиться к БД**

```
org.postgresql.util.PSQLException: Connection to db-svc:5432 refused.
```

Что делать: проверить что БД запущена (`kubectl get pods -l app=postgres`), проверить Service (`kubectl get svc db-svc`), проверить переменные окружения (`kubectl describe pod ... | grep -A 20 Environment`).

**Причина 2: Неправильные переменные окружения**

```
java.lang.IllegalArgumentException: Could not resolve placeholder 'SPRING_DATASOURCE_URL'
```

Что делать: `kubectl describe pod ... | grep -A 30 Environment` — посмотреть что реально передано.

**Причина 3: Не хватает памяти**

```bash
kubectl describe pod order-service-7d8f9b5c6-abc12
# ...
# Last State: Terminated
#   Reason: OOMKilled          ← ВОТ ЭТО
#   Exit Code: 137
```

`OOMKilled` — Kubernetes убил контейнер, потому что он превысил `resources.limits.memory`. Spring Boot с Hibernate может легко сожрать 512МБ. Если стоит limit 256Mi — будет OOMKilled.

Что делать: увеличить limits в манифесте.

**Причина 4: readinessProbe/livenessProbe не проходит**

```
Events:
  Warning  Unhealthy  5s (x3 over 15s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  5s                kubelet  Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Killing    5s                kubelet  Container failed liveness probe, will be restarted
```

Spring Boot стартует 30-40 секунд. Если `initialDelaySeconds` в liveness probe = 10 секунд — Kubernetes решит что приложение мертво и убьёт его **до того как оно стартовало**. Бесконечный цикл: старт → не успел → убит → старт → не успел → убит.

Что делать: увеличить `initialDelaySeconds`. Для Spring Boot:

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30     # ← не меньше, для жирных приложений 60
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60     # ← БОЛЬШЕ чем readiness, дай приложению стартовать
  periodSeconds: 30
```

> **Золотое правило**: `livenessProbe.initialDelaySeconds` > `readinessProbe.initialDelaySeconds`. Readiness говорит «не шли мне трафик пока». Liveness говорит «убей меня если я мёртв». Если liveness проверяет раньше readiness — он убьёт приложение до того, как оно успеет стать ready.

### Задача №3: «Задеплой свой сервис»

Нормальный процесс в компании:

1. Ты пушишь код в Git.
2. CI/CD собирает Docker-образ и пушит в registry.
3. CI/CD обновляет образ в Kubernetes.

Ты не делаешь `kubectl apply` руками на production. Это делает pipeline. Но ты **должен понимать**, что pipeline делает, и уметь сделать это руками на staging.

Типичный pipeline (GitLab CI):

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t registry.company.com/order-service:$CI_COMMIT_SHA .
    - docker push registry.company.com/order-service:$CI_COMMIT_SHA

deploy-staging:
  stage: deploy
  script:
    - kubectl set image deployment/order-service \
        app=registry.company.com/order-service:$CI_COMMIT_SHA \
        -n staging
    - kubectl rollout status deployment/order-service -n staging --timeout=300s
  environment:
    name: staging

deploy-production:
  stage: deploy
  script:
    - kubectl set image deployment/order-service \
        app=registry.company.com/order-service:$CI_COMMIT_SHA \
        -n orders-team
    - kubectl rollout status deployment/order-service -n orders-team --timeout=300s
  environment:
    name: production
  when: manual    # ← production деплой только вручную (кнопка в GitLab)
```

`kubectl set image` — обновляет образ в Deployment. Kubernetes делает rolling update: создаёт новый Pod с новым образом, ждёт readiness, убивает старый.

`kubectl rollout status` — ждёт пока деплой завершится. Если Pod'ы не стартуют за 300 секунд — pipeline упадёт.

### Задача №4: «Откати, что-то сломалось»

Задеплоил → пользователи жалуются → нужно откатить. Быстро.

```bash
# Откат к предыдущей версии
kubectl rollout undo deployment/order-service

# Проверить что откат прошёл
kubectl rollout status deployment/order-service

# Посмотреть историю деплоев
kubectl rollout history deployment/order-service
```

Это **единственная команда, которую ты должен уметь выполнить с закрытыми глазами**. Серьёзно. Если на production что-то пошло не так — откатывай сначала, разбирайся потом. Каждая секунда с багом — это реальные пользователи, которые не могут оплатить заказ, не могут вызвать такси, не могут перевести деньги.

**Реальная история**: пятница, 18:00, задеплоили «маленький фикс». Через 5 минут алерт — error rate 30%. Разработчик начал разбираться в логах, искать причину. 15 минут прошло, error rate 30%, бизнес теряет деньги. Пришёл тимлид: «Откати. Сейчас. Разбираться будем в понедельник». Один `rollout undo` — и через 30 секунд всё работает.

Правило: **откатывай, потом разбирайся**. Не наоборот.

### Задача №5: «Почему мой сервис тормозит?»

```bash
# Сколько ресурсов реально ест Pod
kubectl top pods -l app=order-service
```

```
NAME                             CPU(cores)   MEMORY(bytes)
order-service-7d8f9b5c6-abc12   450m         478Mi
order-service-7d8f9b5c6-def34   380m         465Mi
order-service-7d8f9b5c6-ghi56   920m         490Mi
```

Третий Pod ест 920m CPU при лимите 1000m. Он упирается в потолок. Kubernetes начинает **throttling** — искусственно замедляет Pod. Отсюда и тормоза.

Что делать:
1. Увеличить CPU limits (быстрый фикс).
2. Профилировать приложение — почему оно ест столько CPU (правильный фикс).
3. Масштабировать — больше Pod'ов (если нагрузка растёт).

```bash
# Посмотреть настройки автомасштабирования
kubectl get hpa order-service

# Ручное масштабирование (если нет HPA)
kubectl scale deployment/order-service --replicas=5
```

### Задача №6: «Залезть внутрь Pod'а»

Иногда нужно посмотреть что происходит «изнутри». Может, проверить DNS, может, дёрнуть curl к другому сервису.

```bash
# Зайти в работающий Pod
kubectl exec -it order-service-7d8f9b5c6-abc12 -- bash

# Если bash нет (Alpine-образ)
kubectl exec -it order-service-7d8f9b5c6-abc12 -- sh
```

Внутри Pod'а:

```bash
# Проверить DNS — видит ли Pod другой сервис
nslookup catalog-service
# Server:    10.96.0.10
# Name:      catalog-service.orders-team.svc.cluster.local
# Address:   10.96.45.67

# Дёрнуть другой сервис
curl http://catalog-service:8080/actuator/health
# {"status":"UP"}

# Посмотреть переменные окружения
env | grep SPRING
# SPRING_DATASOURCE_URL=jdbc:postgresql://db-svc:5432/orders
# SPRING_PROFILES_ACTIVE=production

# Проверить, что файл конфигурации на месте (если монтировали через ConfigMap)
cat /etc/config/application.yml
```

> **Важно**: в production-образах часто нет curl, nslookup и даже bash. Образ минимальный — только JRE + JAR. Если нужно отлаживать сеть, используй **ephemeral debug container**:

```bash
kubectl debug -it order-service-7d8f9b5c6-abc12 --image=busybox:latest --target=app
```

Это запустит busybox (в котором есть curl, nslookup, ping) рядом с твоим контейнером, в том же сетевом пространстве.

### Задача №7: «Добавить переменную окружения / изменить конфигурацию»

Добавил новую feature flag? Нужен новый ключ в ConfigMap.

```bash
# Посмотреть текущий ConfigMap
kubectl get configmap order-service-config -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  SPRING_PROFILES_ACTIVE: "production"
  ORDER_PROCESSING_TIMEOUT: "30s"
  FEATURE_NEW_CHECKOUT: "false"           # ← хочешь поменять на true
```

```bash
# Отредактировать
kubectl edit configmap order-service-config
# Откроется vim. Меняешь "false" → "true". Сохраняешь.
```

Но **Pod не подхватит изменение автоматически**! ConfigMap обновился, но Pod читает переменные окружения при старте. Нужно перезапустить Pod'ы:

```bash
# Мягкий перезапуск (rolling restart — без простоя)
kubectl rollout restart deployment/order-service
```

Это убьёт Pod'ы по одному и создаст новые. Новые подхватят обновлённый ConfigMap.

---

## Часть 3. Resources — самая важная настройка, которую все делают неправильно

### Почему это важно

Вот кусок манифеста, который определяет, будет ли твой сервис работать стабильно или будет случайно умирать:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**requests** — «мне нужно минимум столько». Scheduler использует requests, чтобы решить, на какой узел поставить Pod. Если на узле нет свободных 256Mi — Pod туда не попадёт.

**limits** — «мне нельзя больше». Если Pod превысит memory limit — **OOMKilled** (убит). Если превысит CPU limit — **throttling** (замедлен, но не убит).

### Типичные ошибки

**Ошибка 1: Не указать resources вообще**

```yaml
# ❌ Нет resources
containers:
  - name: app
    image: order-service:1.0
```

Pod может сожрать **всю память узла**. Один твой сервис с утечкой памяти роняет все Pod'ы на узле. Включая сервисы других команд.

В нормальных компаниях есть LimitRange или Admission Controller, который не пустит Pod без resources. Но не везде.

**Ошибка 2: Слишком маленький memory limit**

```yaml
# ❌ Spring Boot с Hibernate в 256Mi?
resources:
  limits:
    memory: "256Mi"
```

Spring Boot + Hibernate + connection pool + пара библиотек = 400-600 МБ. С лимитом 256Mi Pod будет постоянно OOMKilled.

Как узнать сколько нужно? Запусти на staging и посмотри:

```bash
kubectl top pods -l app=order-service
```

Возьми пиковое значение и добавь 20-30% запаса.

**Ошибка 3: requests = limits**

```yaml
# ⚠️ Жёстко — работает, но расточительно
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

Kubernetes зарезервирует 512Mi на узле, даже если Pod реально ест 200Mi. Узлы будут «заполнены» по бумагам, но пустые по факту. Дорого.

**Рекомендация для старта:**

```yaml
resources:
  requests:
    memory: "256Mi"     # реальное среднее потребление
    cpu: "100m"          # 0.1 ядра
  limits:
    memory: "512Mi"     # requests × 2
    cpu: "500m"          # requests × 5 (CPU throttling не убивает)
```

Потом подкрутишь на основе реальных данных с `kubectl top`.

> **Что такое "m" в CPU**: 1000m = 1 ядро. 100m = 10% одного ядра. 500m = половина ядра.

---

## Часть 4. Probes — почему твой сервис перезапускается без причины

### Три типа probes

```yaml
# Готов ли принимать трафик?
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

# Жив ли вообще?
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 3

# Стартовал ли? (Kubernetes 1.20+)
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30        # 30 × 5 = 150 секунд на старт
```

### Чем они отличаются

**startupProbe** — проверяет при **старте**. Пока startup probe не пройдёт, liveness и readiness **не запускаются**. Идеален для медленно стартующих приложений. `failureThreshold: 30` × `periodSeconds: 5` = 150 секунд на старт. Если за это время приложение не ответит — Pod будет убит.

**readinessProbe** — проверяет **готовность принимать трафик**. Если не прошла — Pod остаётся живым, но Service **перестаёт направлять** к нему запросы. Полезно: приложение запустилось, но ещё грузит кэш. Или временно перегружено. Не надо его убивать — просто не шли ему трафик.

**livenessProbe** — проверяет **жизнь**. Если не прошла — Kubernetes **убивает** Pod и создаёт новый. Это крайняя мера. Используй когда приложение может зависнуть намертво (deadlock, бесконечный цикл).

### Распространённая ошибка

```yaml
# ❌ Одинаковый путь и одинаковые таймауты для всех probes
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 5       # ← приложение ещё стартует!
  periodSeconds: 5
```

Spring Boot стартует 30-40 секунд. Через 5 секунд liveness probe стучит → не отвечает → `failureThreshold: 3` × 5 сек = 15 секунд → Pod убит. Приложение даже не успело стартовать. Перезапуск. Снова 5 сек → не отвечает → убит. **CrashLoopBackOff**.

**Правильная конфигурация для Spring Boot:**

```yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30          # 150 секунд на старт — достаточно

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 30
  failureThreshold: 3
```

Spring Boot Actuator с версии 2.3 имеет отдельные эндпоинты `/health/readiness` и `/health/liveness`. Включаются так:

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

---

## Часть 5. Отладка сети — «сервис не видит другой сервис»

Это будет происходить регулярно. Чаще всего — при добавлении нового сервиса.

### Алгоритм

```bash
# 1. Проверить что целевой Pod работает
kubectl get pods -l app=catalog-service
# Если Pod'ов нет — вот и причина

# 2. Проверить что Service существует
kubectl get svc catalog-service
# Если Service нет — создай

# 3. Проверить что Service видит Pod'ы (endpoints)
kubectl get endpoints catalog-service
# ENDPOINTS
# 10.244.0.5:8080,10.244.1.3:8080

# Если ENDPOINTS пустые — labels не совпадают!
# Это САМАЯ ЧАСТАЯ ошибка.
```

**Реальная история**: потратил час на дебаг «order-service не может достучаться до catalog-service». Все Pod'ы Running, Service создан, DNS резолвит. Но endpoints пустые. Оказалось:

```yaml
# Deployment
metadata:
  labels:
    app: catalog-service     # ← с дефисом

# Service
spec:
  selector:
    app: catalog_service     # ← с подчёркиванием!!! 
```

Одна буква. Час жизни. Service ищет Pod'ы по label'ам, и если label в Deployment не совпадает с selector в Service — endpoints будут пустые. Service не найдёт Pod'ы.

```bash
# 4. Проверить DNS изнутри Pod'а
kubectl exec -it order-service-abc12 -- nslookup catalog-service
# Должен вернуть IP. Если "can't resolve" — проблема в DNS или имени

# 5. Проверить связь изнутри Pod'а
kubectl exec -it order-service-abc12 -- curl http://catalog-service:8080/actuator/health
# Если таймаут — NetworkPolicy блокирует или порт неправильный
```

---

## Часть 6. Секреты и конфиги — как не захардкодить пароль

### Правило

```
❌ В коде:          spring.datasource.password=MyS3cretP@ss
❌ В application.yml: spring.datasource.password=MyS3cretP@ss  
❌ В Git:           Закоммитил .env с паролями

✅ В Kubernetes Secret, передано через env
```

### Как создают секреты

```bash
# Руками (для staging)
kubectl create secret generic order-db-secret \
  --from-literal=DB_USERNAME=order_user \
  --from-literal=DB_PASSWORD='MyS3cretP@ss'

# Через CI/CD (для production)
# Пароль хранится в GitLab Variables / GitHub Secrets / Vault
# Pipeline создаёт Secret автоматически
```

### Как используют в Deployment

```yaml
env:
  - name: SPRING_DATASOURCE_USERNAME
    valueFrom:
      secretKeyRef:
        name: order-db-secret
        key: DB_USERNAME
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-db-secret
        key: DB_PASSWORD
```

### Как проверить, что передано

```bash
# Посмотреть какие env реально видит Pod
kubectl exec -it order-service-abc12 -- env | grep SPRING_DATASOURCE

# Или через describe (покажет ссылки на Secret, но не значения)
kubectl describe pod order-service-abc12 | grep -A 5 Environment
```

> **Никогда** не делай `kubectl get secret -o yaml` на production и не кидай вывод в чат/Slack. base64 — это **не шифрование**. `echo "cGFzc3dvcmQ=" | base64 -d` → `password`. Любой прочитает.

---

## Часть 7. Полезные приёмы, которые экономят время

### Алиасы

Добавь в `~/.bashrc` или `~/.zshrc`:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kd='kubectl describe'
alias ke='kubectl exec -it'
```

После этого: `kgp` вместо `kubectl get pods`. Экономит тысячи нажатий в год.

### kubectl контексты

Если работаешь с несколькими кластерами (staging + production):

```bash
# Посмотреть все контексты
kubectl config get-contexts

# Переключиться
kubectl config use-context staging-cluster

# ВСЕГДА проверяй перед опасными операциями
kubectl config current-context
```

**Реальная история**: разработчик хотел удалить тестовый Deployment на staging. Не проверил контекст. Удалил на production. Сервис лежал 4 минуты, пока не пересоздали. Теперь у нас в команде правило: перед любым `delete` — `kubectl config current-context`.

### Stern — логи нескольких Pod'ов

`kubectl logs` показывает логи одного Pod'а. Если реплик три — неудобно. Stern (отдельная утилита) показывает логи всех Pod'ов по паттерну, с цветами:

```bash
# Установка
brew install stern       # macOS
# или скачать бинарник с GitHub

# Логи всех Pod'ов order-service
stern order-service

# Логи всех Pod'ов в namespace
stern . -n orders-team
```

### k9s — терминальный UI

```bash
brew install k9s
k9s
```

Терминальный интерфейс для Kubernetes. Показывает Pod'ы, логи, describe — всё в одном окне. Навигация стрелками. Многие опытные разработчики используют именно k9s, а не голый kubectl.

### Быстрая проверка YAML перед apply

```bash
# Dry-run: покажет что будет создано, но не создаст
kubectl apply -f deployment.yaml --dry-run=client

# Валидация на стороне сервера (проверит схему)
kubectl apply -f deployment.yaml --dry-run=server
```

---

## Часть 8. Чек-лист: перед тем как идти на работу

Распечатай и повесь рядом с монитором:

```
СЕРВИС НЕ РАБОТАЕТ:
  1. kubectl get pods -l app=<name>         ← Pod вообще есть?
  2. kubectl describe pod <name>            ← Events + статус
  3. kubectl logs <name> --previous         ← логи упавшего контейнера
  4. kubectl get endpoints <service-name>   ← Service видит Pod'ы?

ЗАДЕПЛОИЛ И СЛОМАЛОСЬ:
  1. kubectl rollout undo deployment/<name> ← ОТКАТИ СНАЧАЛА
  2. kubectl logs -l app=<name> --previous  ← потом разбирайся

ТОРМОЗИТ:
  1. kubectl top pods -l app=<name>         ← CPU/Memory
  2. kubectl describe hpa <name>            ← автоскейлер
  3. kubectl logs -l app=<name> --since=5m  ← ошибки?

НЕ ВИДИТ ДРУГОЙ СЕРВИС:
  1. kubectl get endpoints <target-service> ← endpoints пустые?
  2. kubectl exec -it <pod> -- curl <target-service>:<port>/health
  3. Проверь labels: Deployment vs Service selector

ПЕРЕД ДЕПЛОЕМ НА PRODUCTION:
  1. kubectl config current-context          ← ТОТ ЛИ КЛАСТЕР?
  2. Протестировал на staging?
  3. Мониторинг открыт? (Grafana)
  4. Знаешь как откатить? (kubectl rollout undo)
```

---

## Заключение

Вот и всё. Не бойся Kubernetes — он большой, но твоя ежедневная работа с ним укладывается в 10-15 команд. Остальное нарастёт с практикой.

Главное помни три вещи:

1. **Откатывай, потом разбирайся.** Каждая минута с багом на production — это реальные потери.

2. **Проверяй контекст.** Перед каждой мутирующей операцией (`delete`, `apply`, `scale`) — `kubectl config current-context`. Это не паранойя, это профессионализм.

3. **Не стесняйся спрашивать.** Kubernetes огромный. Никто не знает его целиком. Даже Senior с 5 годами опыта гуглит «kubectl cheat sheet». Это нормально.

Удачи, парнишка. Ты справишься.
