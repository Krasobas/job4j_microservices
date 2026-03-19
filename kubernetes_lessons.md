# 3.6.6. Kubernetes — уроки (2026)

> **Целевой стек**: Kubernetes 1.35, minikube 1.38, kubectl, Helm 3, Java 21, Spring Boot 3.4
>
> **Обозначения**: 🪟 Windows · 🍎 macOS · 🐧 Linux
>
> **Предпосылки**: пройден раздел 3.6.5 Docker (умеете собирать образы, писать Dockerfile, работать с Docker Compose)

---

## Урок 1. Зачем Kubernetes

### Проблема

Docker Compose позволяет запускать несколько контейнеров на **одной** машине. Для учебных проектов и небольших систем этого достаточно. Но в production-среде возникают задачи, которые Docker Compose не решает:

- **Масштабирование** — нужно запустить 10 копий одного сервиса и распределить нагрузку между ними.
- **Отказоустойчивость** — если контейнер упал, его надо автоматически перезапустить. Если упал весь сервер — перенести контейнеры на другой.
- **Работа на нескольких серверах** — Docker Compose ограничен одной машиной.
- **Zero-downtime деплой** — обновить сервис без простоя.
- **Service Discovery** — сервисы должны находить друг друга без хардкода IP-адресов.

### Масштабирование: вглубь vs вширь

Раньше, когда приложению не хватало мощности, ответ был один: купить сервер помощнее. Больше RAM, быстрее CPU. Это **масштабирование вглубь** (vertical scaling). Проблема — рост цены нелинейный (сервер в 2 раза мощнее стоит в 3–5 раз дороже), и есть физический потолок.

Альтернатива — **масштабирование вширь** (horizontal scaling): взять несколько обычных серверов и распределить нагрузку между ними. Дешевле, потолка практически нет, но возникают вопросы: как распределить трафик? Что если один сервер упал? Как синхронизировать данные?

Именно эти вопросы и решает Kubernetes.

```
Вглубь (vertical):          Вширь (horizontal):

  ┌─────────────┐            ┌───────┐ ┌───────┐ ┌───────┐
  │             │            │Server │ │Server │ │Server │
  │  Мощный     │            │  1    │ │  2    │ │  3    │
  │  сервер     │            │       │ │       │ │       │
  │  $$$$$      │            │  $    │ │  $    │ │  $    │
  │             │            └───┬───┘ └───┬───┘ └───┬───┘
  └─────────────┘                └─────────┼─────────┘
                                     Load Balancer
                                     (Kubernetes)
```

### Stateless vs Stateful

Чтобы масштабирование вширь работало, приложение должно быть **Stateless** — не хранить состояние внутри себя. Любой запрос может прийти на любую копию, и результат должен быть одинаковым.

- **Stateless** — REST API, бизнес-логика, gateway. Можно создать 10 копий — каждая обрабатывает запросы независимо. Идеально для масштабирования.
- **Stateful** — база данных, кэш, брокер сообщений. Состояние хранится на диске. Нельзя просто создать 5 копий PostgreSQL — данные разъедутся. Для таких приложений в K8s есть отдельный механизм — **StatefulSet** (подробнее в уроке 8).

> Ваш Spring Boot REST API — stateless. Сессии не хранятся на сервере (используется JWT). Это значит, что его легко масштабировать: `replicas: 10` — и Kubernetes создаст 10 одинаковых Pod'ов.

### Что такое Kubernetes

Kubernetes (K8s) — платформа для **оркестрации контейнеров**. Она автоматизирует развёртывание, масштабирование и управление контейнеризированными приложениями на кластере из одного или нескольких серверов.

Ключевое слово — **декларативность**. Вместо того чтобы писать скрипты «запусти контейнер А, потом Б», вы описываете **желаемое состояние**: «хочу 3 копии сервиса А, с доступом на порту 8080, с 512МБ памяти». Kubernetes сам приводит кластер к этому состоянию и поддерживает его.

### Docker Compose vs Kubernetes

| Критерий | Docker Compose | Kubernetes |
|---------|---------------|-----------|
| Где запускается | Одна машина | Кластер (1..N машин) |
| Масштабирование | Вручную | Автоматическое (HPA) |
| Самовосстановление | `restart: always` | Полноценный: перезапуск, перенос на другой узел |
| Балансировка нагрузки | Нет (нужен nginx руками) | Встроенная (Service) |
| Zero-downtime деплой | Нет | Rolling update из коробки |
| Сложность | Минимальная | Высокая |
| Когда использовать | Разработка, малые проекты | Production, микросервисы |

### Kubernetes в цифрах (2026)

- 92% организаций используют контейнеры в production, Kubernetes — доминирующий оркестратор (CNCF Survey 2025).
- Более 5.6 млн разработчиков используют Kubernetes.
- 200+ сертифицированных дистрибутивов и платформ.
- Текущая стабильная версия: **1.35** (релизный цикл ~4 месяца).

### Managed Kubernetes

В production-среде Kubernetes чаще используют как **managed-сервис** от облачного провайдера:

- **AWS** — Amazon EKS (Elastic Kubernetes Service)
- **Google Cloud** — GKE (Google Kubernetes Engine)
- **Azure** — AKS (Azure Kubernetes Service)
- **Yandex Cloud** — Managed Service for Kubernetes

Managed-кластер снимает с вас задачи по настройке и обслуживанию control plane. Вы управляете только рабочими узлами и приложениями.

Для обучения и локальной разработки мы будем использовать **minikube** — лёгкий Kubernetes-кластер на локальной машине.

### Задание

Ознакомьтесь с материалом урока. Убедитесь, что понимаете, какие задачи решает Kubernetes и чем он отличается от Docker Compose.

---

## Урок 2. Установка

Для работы с Kubernetes нам нужны два инструмента:

- **minikube** — создаёт локальный Kubernetes-кластер (один узел) на вашей машине.
- **kubectl** — CLI для управления Kubernetes-кластером (любым, не только minikube).

### Предпосылки

Должен быть установлен Docker (из раздела 3.6.5).

### Установка kubectl

kubectl — клиент командной строки для Kubernetes. Версия kubectl должна отличаться от версии кластера не более чем на ±1 минорную версию.

**🪟 Windows (PowerShell от администратора):**

```powershell
# Через Chocolatey
choco install kubernetes-cli

# Или через winget
winget install -e --id Kubernetes.kubectl
```

**🍎 macOS:**

```bash
brew install kubectl
```

**🐧 Linux:**

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

**Проверка:**

```bash
kubectl version --client
```

### Установка minikube

**🪟 Windows (PowerShell от администратора):**

```powershell
# Через Chocolatey
choco install minikube

# Или через winget
winget install -e --id Kubernetes.minikube
```

**🍎 macOS:**

```bash
brew install minikube
```

**🐧 Linux (x86-64):**

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

### Запуск кластера

```bash
minikube start --driver=docker
```

Флаг `--driver=docker` говорит minikube использовать Docker как среду для запуска кластера. Это самый простой и рекомендуемый вариант.

Первый запуск может занять несколько минут — minikube скачает образ Kubernetes.

Вывод при успешном запуске:

```
😄  minikube v1.38.x on ...
✨  Using the docker driver based on user configuration
📦  Preparing Kubernetes v1.35.x on Docker ...
🏄  Done! kubectl is now configured to use "minikube" cluster
```

### Проверка

```bash
# Статус кластера
minikube status

# Информация о кластере через kubectl
kubectl cluster-info

# Список узлов (будет один — minikube)
kubectl get nodes
```

### Остановка и удаление

```bash
# Остановить кластер (данные сохраняются)
minikube stop

# Запустить снова
minikube start

# Полностью удалить кластер
minikube delete
```

### Альтернатива: Docker Desktop Kubernetes

Docker Desktop (Windows / macOS) имеет встроенный Kubernetes. Включается в Settings → Kubernetes → Enable Kubernetes. Это удобно, если не хотите ставить minikube отдельно. Но minikube гибче: поддерживает несколько версий K8s, аддоны (dashboard, ingress), мультинодовые кластеры.

### minikube dashboard

minikube поставляется с веб-интерфейсом Kubernetes Dashboard:

```bash
minikube dashboard
```

Эта команда откроет браузер с GUI, где можно просматривать pods, deployments, services и т.д. Удобно для обучения.

### Задание

1. Установите kubectl и minikube.
2. Запустите кластер: `minikube start --driver=docker`.
3. Выполните `kubectl get nodes` и `minikube status`.
4. Откройте minikube dashboard.
5. Приложите скриншоты к заданию.

---

## Урок 3. Архитектура Kubernetes

Прежде чем создавать ресурсы, нужно понимать, из чего состоит кластер.

### Структура кластера

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                          │
│                                                                 │
│  ┌───────────────────── Control Plane ──────────────────────┐   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌────────┐  │   │
│  │  │ API      │  │ Scheduler │  │Controller│  │  etcd  │  │   │
│  │  │ Server   │  │           │  │ Manager  │  │        │  │   │
│  │  └────┬─────┘  └───────────┘  └──────────┘  └────────┘  │   │
│  │       │                                                  │   │
│  └───────┼──────────────────────────────────────────────────┘   │
│          │ kubectl / API                                        │
│  ┌───────┼──────────────────────────────────────────────────┐   │
│  │       │            Worker Node 1                         │   │
│  │  ┌────┴─────┐                                            │   │
│  │  │ kubelet  │  ┌───────┐  ┌───────┐  ┌───────┐          │   │
│  │  ├──────────┤  │ Pod A │  │ Pod B │  │ Pod C │          │   │
│  │  │kube-proxy│  └───────┘  └───────┘  └───────┘          │   │
│  │  └──────────┘                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Worker Node 2                           │   │
│  │  ┌──────────┐                                            │   │
│  │  │ kubelet  │  ┌───────┐  ┌───────┐                      │   │
│  │  ├──────────┤  │ Pod D │  │ Pod E │                      │   │
│  │  │kube-proxy│  └───────┘  └───────┘                      │   │
│  │  └──────────┘                                            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Control Plane (управляющий уровень)

Компоненты, которые принимают решения о кластере:

**API Server** (`kube-apiserver`) — входная точка для всех запросов.

```
Вы: kubectl apply -f deployment.yaml
     └──▶ API Server принимает, валидирует YAML, сохраняет в etcd

Вы: kubectl get pods
     └──▶ API Server читает из etcd и возвращает вам список
```

Любое взаимодействие с кластером — через API Server. kubectl, Helm, CI/CD, дашборд — все стучатся в него.

**etcd** — распределённое key-value хранилище. Хранит **всё состояние** кластера.

```
После kubectl apply:
  etcd хранит: {"kind": "Deployment", "name": "order-service", "replicas": 3, ...}

После создания Pod'а:
  etcd хранит: {"kind": "Pod", "name": "order-service-abc12", "node": "worker-1", ...}
```

Если etcd умер — кластер «забыл» всё: какие Deployment'ы, сколько реплик, какие Service. Pod'ы продолжат работать (они уже запущены), но управлять кластером нельзя.

**Scheduler** (`kube-scheduler`) — решает, **на каком узле** запустить новый Pod.

```
Scheduler видит: новый Pod без назначенного узла
  ├── Worker-1: CPU 80% занято, RAM 70%
  ├── Worker-2: CPU 30% занято, RAM 40%  ← свободнее
  └── Worker-3: CPU 60% занято, RAM 50%

Scheduler решает: Pod → Worker-2
Записывает в etcd: Pod.node = "worker-2"
```

**Controller Manager** — набор роботов-контроллеров, каждый следит за «своим» типом ресурса.

```
Deployment Controller непрерывно проверяет:
  "replicas: 3, а Pod'ов сейчас 2. Создаю ещё один."
  "Pod упал → осталось 2 из 3. Создаю замену."
  "Образ обновился → rolling update: создаю новый Pod, жду ready, убиваю старый."
```

### Worker Node (рабочий узел)

Машина (физическая или виртуальная), на которой запускаются контейнеры:

**kubelet** — агент на каждом узле. Получает указания от API Server и управляет контейнерами.

```
Scheduler назначил Pod на этот узел → kubelet видит:
  "Мне назначили Pod order-service-abc12"
  "Скачиваю образ order-service:1.0..."
  "Запускаю контейнер..."
  "Проверяю readinessProbe... ОК, Pod Ready."
  "Каждые 30 сек проверяю livenessProbe... жив."
```

**kube-proxy** — настраивает сетевые правила для маршрутизации трафика к Pod'ам.

```
Создан Service order-svc (ClusterIP: 10.96.0.15, port: 8080)
За ним 3 Pod'а: 10.244.0.5, 10.244.1.3, 10.244.0.9

kube-proxy на КАЖДОМ узле настраивает правила:
  "Запрос на 10.96.0.15:8080 → отправить на один из трёх Pod'ов"
  + регистрирует DNS: order-svc → 10.96.0.15
```

**Container Runtime** — среда выполнения контейнеров. containerd — стандарт с K8s 1.24+. Docker как runtime удалён (образы, собранные Docker, работают — они OCI-совместимы, но сам Docker daemon больше не используется K8s).

> **Важно**: В minikube Control Plane и Worker Node совмещены на одном узле. В production это обычно разные машины.

### Как устроен Kubernetes: база данных + роботы

Чтобы понять Kubernetes, забудьте про YAML-файлы на минуту. Вот вся суть:

**Kubernetes — это база данных (etcd) и набор роботов (контроллеров), которые непрерывно смотрят в неё и делают свою работу.**

```
┌──────────────────────────────────────────────────────────┐
│                      etcd                                │
│              (распределённая БД)                         │
│                                                          │
│  Хранит ВСЕ ресурсы как JSON-документы:                  │
│                                                          │
│  {"kind": "Deployment", "name": "hello-app", ...}        │
│  {"kind": "Service", "name": "hello-app-svc", ...}       │
│  {"kind": "Ingress", "name": "hello-app-ingress", ...}   │
│  {"kind": "Pod", "name": "hello-app-7d8f9b5c6-abc12",...}│
│                                                          │
└──────────────────────────┬───────────────────────────────┘
                           │
                           │  Роботы (контроллеры) непрерывно
                           │  смотрят в etcd и делают свою работу
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌───────────┐  ┌──────────────┐
   │ Deployment  │  │  Service  │  │   Ingress    │
   │ Controller  │  │ Controller│  │  Controller   │
   │  "Робот"    │  │  "Робот"  │  │   "Робот"    │
   └─────────────┘  └───────────┘  └──────────────┘
```

Когда вы пишете YAML и делаете `kubectl apply` — вы **не отдаёте команду**. Вы кладёте **документ** в базу данных. А роботы сами замечают и делают работу.

### Что происходит при `kubectl apply`

```
Вы:  kubectl apply -f deployment.yaml

                │
                ▼
          ┌───────────┐
          │ API Server │   принял YAML, провалидировал,
          └─────┬─────┘   сохранил в etcd как JSON-документ
                │
                ▼
          ┌───────────┐
          │   etcd     │   теперь там лежит запись:
          │            │   {kind: Deployment, replicas: 3, image: nginx:1.27}
          └─────┬──────┘
                │
                │   Deployment Controller непрерывно смотрит в etcd.
                │   Увидел новый Deployment.
                ▼
          ┌────────────────────┐
          │Deployment Controller│  "О, новый Deployment! replicas: 3.
          │                    │   Сейчас Pod'ов: 0. Нужно создать 3."
          │                    │
          │                    │   Создаёт 3 записи Pod в etcd.
          └────────┬───────────┘
                   │
                   ▼
          ┌────────────────────┐
          │    Scheduler       │  "О, 3 новых Pod'а без узла!
          │                    │   Node 1 свободнее — Pod 1 туда.
          │                    │   Node 2 — Pod 2 и 3 туда."
          │                    │
          │                    │   Записывает в etcd: Pod 1 → Node 1
          └────────┬───────────┘
                   │
                   ▼
          ┌────────────────────┐
          │    kubelet         │  "О, мне назначили Pod 1!
          │    (на Node 1)     │   Скачиваю образ nginx:1.27...
          │                    │   Запускаю контейнер...
          │                    │   Готово."
          └────────────────────┘
```

Цепочка: **Вы** написали YAML → **API Server** сохранил в etcd → **Робот** увидел → сделал работу → записал результат обратно → следующий робот увидел → сделал свою часть.

### Каждый ресурс — задание для своего робота

Когда вы пишете YAML с `kind: Deployment` или `kind: Service` — вы создаёте задание для конкретного робота:

| Вы пишете в YAML | Робот (контроллер) | Что робот делает |
|------------------|--------------------|-----------------|
| `kind: Deployment` | Deployment Controller | Создаёт/удаляет Pod'ы, следит что их ровно `replicas` штук, делает rolling update |
| `kind: Service` | kube-proxy (на каждом узле) | Настраивает сетевые правила: «запрос на ClusterIP → направь на один из Pod'ов». Регистрирует DNS. |
| `kind: Ingress` | Ingress Controller (Pod с nginx) | Перенастраивает nginx: «Host: myapp.com, path /api → forward на hello-app-svc:8080» |
| `kind: ConfigMap` | Нет робота | Просто лежит в etcd. Pod'ы читают при старте. |
| `kind: Secret` | Нет робота | Просто лежит в etcd. Pod'ы читают при старте. |
| `kind: PVC` | PV Controller | Находит подходящий диск (или просит облако создать) и привязывает к Pod'у. |

### Непрерывный цикл (reconciliation loop)

Роботы не выполняют задание один раз и засыпают. Они работают **в бесконечном цикле**:

```
while true:
    desired = прочитать из etcd "что хочет пользователь"
    actual  = посмотреть "что сейчас реально"

    if actual != desired:
        сделать действия чтобы actual стал как desired

    sleep(немного)
```

Поэтому:
- Удалил Pod руками → Controller видит: «хотели 3, осталось 2» → создаёт ещё один.
- Pod упал → Controller видит расхождение → создаёт новый.
- Поменял `replicas: 5` → Controller видит: «хотели 5, есть 3» → создаёт ещё 2.

Вы не командуете «создай Pod». Вы говорите «я хочу 3 Pod'а». Робот сам разбирается как этого достичь и **поддерживает это состояние вечно**.

### Аналогия

Представьте кондиционер. Вы не командуете: «включи компрессор на 70%, открой заслонку на 30%». Вы ставите **желаемую температуру** (22°C). Кондиционер сам непрерывно сравнивает текущую и желаемую температуру и делает что нужно.

Kubernetes — тот же принцип. YAML-файлы — это термостат. Контроллеры — это кондиционер.

### Основные ресурсы Kubernetes

| Ресурс | Назначение | Аналог в Docker |
|--------|-----------|----------------|
| **Pod** | Минимальная единица запуска (один или несколько контейнеров) | `docker run` |
| **Deployment** | Управляет Pod'ами: количество реплик, обновления | — |
| **Service** | Стабильная сетевая точка доступа к Pod'ам | `ports` в Compose |
| **ConfigMap** | Конфигурация (key-value) | `environment` в Compose |
| **Secret** | Секреты (пароли, ключи) | `environment` (но зашифровано) |
| **PersistentVolume** | Постоянное хранилище | `volumes` в Compose |
| **Ingress** | HTTP-маршрутизация (домены, пути) | — |
| **Namespace** | Логическое разделение ресурсов в кластере | — |

### kubectl — основные команды

```bash
# Получить ресурсы определённого типа
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# Подробная информация о ресурсе
kubectl describe pod <name>

# Применить манифест (создать или обновить ресурс)
kubectl apply -f manifest.yaml
kubectl apply -f k8s/              # все YAML-файлы из папки

# Удалить ресурс
kubectl delete -f manifest.yaml
kubectl delete pod <name>

# Логи контейнера в Pod'е
kubectl logs <pod-name>
kubectl logs -f <pod-name>          # follow
kubectl logs -l app=my-app          # по label (удобно, когда Pod'ов несколько)

# Выполнить команду внутри Pod'а
kubectl exec -it <pod-name> -- bash

# Показать все ресурсы в namespace
kubectl get all
```

### Namespace

Namespace — логическое разделение ресурсов в кластере. По умолчанию:
- `default` — ваши ресурсы, если namespace не указан.
- `kube-system` — компоненты самого Kubernetes.
- `kube-public` — публичные ресурсы.

```bash
kubectl get namespaces
kubectl get pods -n kube-system   # pod'ы Kubernetes-системы
```

### Задание

1. Выполните `kubectl get nodes -o wide` — посмотрите информацию об узлах.
2. Выполните `kubectl get pods -n kube-system` — какие системные компоненты работают?
3. Выполните `kubectl get namespaces`.
4. Приложите скриншоты и кратко опишите, что вы видите.

---

## Урок 4. Pod — базовая единица

### Что такое Pod

Pod (произносится «под») — минимальная единица развёртывания в Kubernetes. Pod содержит один или несколько контейнеров, которые:
- разделяют одну сетевую среду (localhost);
- имеют общие тома;
- всегда запускаются на одном узле.

```
┌─────────── Pod ───────────┐
│                           │
│  ┌───────────┐            │
│  │ Container │  ← обычно один контейнер = один Pod
│  │ (app)     │
│  └───────────┘            │
│                           │
│  IP: 10.244.0.5           │
│  Volumes: [shared-data]   │
└───────────────────────────┘
```

> В 95% случаев Pod = один контейнер. Multi-container pod'ы используются для sidecar-паттернов (логирование, прокси, service mesh).

### Создание Pod через YAML

Создайте файл `pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-java
  labels:
    app: hello-java
spec:
  containers:
    - name: java
      image: eclipse-temurin:21-jdk
      command: ["jshell"]
      stdin: true
      tty: true
```

Применяем:

```bash
kubectl apply -f pod.yaml
```

Проверяем:

```bash
# Список pod'ов
kubectl get pods

# Подробная информация
kubectl describe pod hello-java

# Подключаемся к Pod'у
kubectl attach -it hello-java

# Внутри jshell:
System.out.println("Hello from Kubernetes!")
/exit
```

### Анатомия YAML-манифеста

Каждый ресурс Kubernetes описывается четырьмя обязательными полями:

```yaml
apiVersion: v1              # Версия API (v1 для Pod, apps/v1 для Deployment)
kind: Pod                   # Тип ресурса
metadata:                   # Метаданные
  name: hello-java          # Имя (уникальное в namespace)
  labels:                   # Метки (key-value, для селекции)
    app: hello-java
spec:                       # Спецификация (что именно создать)
  containers:
    - name: java
      image: eclipse-temurin:21-jdk
```

### Несколько ресурсов в одном файле

YAML позволяет разделять документы через `---`. Это удобно, когда ресурсы логически связаны — например, Deployment и его Service всегда создаются вместе:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  # ...
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  # ...
```

Одна команда `kubectl apply -f my-app.yaml` создаст оба ресурса. Также можно применить все файлы из папки: `kubectl apply -f k8s/` — Kubernetes обработает все `.yaml`-файлы в директории.

### Labels и Selectors

**Labels** — произвольные метки на ресурсах. Они не влияют на поведение, но позволяют группировать и фильтровать ресурсы.

```yaml
metadata:
  labels:
    app: tracker
    environment: production
    team: backend
```

**Selectors** — выбирают ресурсы по меткам:

```bash
kubectl get pods -l app=tracker
kubectl get pods -l environment=production
kubectl get pods -l app=tracker,team=backend
```

Labels + Selectors — ключевой механизм Kubernetes. Service находит «свои» Pod'ы по label'ам. Deployment управляет Pod'ами по label'ам.

### Pod — не для production

Pod'ы эфемерны. Если Pod упал, никто его не перезапустит. Если узел вышел из строя — Pod потерян. Поэтому **напрямую Pod'ы почти никогда не создают**. Вместо этого используют **Deployment** (следующий урок), который управляет Pod'ами автоматически.

Прямое создание Pod'а через `kind: Pod` используется для:
- Обучения
- Разовых задач (хотя для этого есть Job)
- Отладки

### Полезные команды для работы с Pod'ами

```bash
# Логи
kubectl logs hello-java
kubectl logs -f hello-java          # follow
kubectl logs hello-java --previous  # логи предыдущего (упавшего) контейнера

# Выполнить команду внутри
kubectl exec -it hello-java -- bash
kubectl exec hello-java -- java --version

# Перенаправление порта (port-forward)
kubectl port-forward hello-java 8080:8080
# Теперь localhost:8080 хоста → 8080 Pod'а

# Удалить Pod
kubectl delete pod hello-java
# Или через файл
kubectl delete -f pod.yaml
```

### Задание

1. Создайте Pod с образом `eclipse-temurin:21-jdk` (файл `pod.yaml`). Подключитесь к нему и выполните `java --version`.
2. Создайте Pod с образом `nginx`. Используйте `kubectl port-forward` для доступа к нему с хоста. Откройте http://localhost:8080 в браузере.
3. Удалите Pod с nginx. Убедитесь, что он не перезапустился автоматически (почему?).
4. Приложите скриншоты.

---

## Урок 5. Deployments и ReplicaSets

### Проблема: Pod'ы не восстанавливаются

Если Pod упал или узел вышел из строя, Pod просто исчезает. Нужен механизм, который:

- Следит, чтобы всегда работало нужное количество копий (реплик) Pod'а.
- Автоматически перезапускает упавшие Pod'ы.
- Позволяет обновлять приложение без простоя.

Это **Deployment**.

### Deployment

Deployment — высокоуровневый ресурс, который управляет ReplicaSet, который, в свою очередь, управляет Pod'ами:

```
Deployment
  └─ ReplicaSet
       ├─ Pod 1
       ├─ Pod 2
       └─ Pod 3
```

Создайте файл `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
        - name: app
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
```

Разберём ключевые поля:

- **replicas: 3** — Kubernetes будет поддерживать ровно 3 запущенных Pod'а.
- **selector.matchLabels** — по какому label'у Deployment находит «свои» Pod'ы.
- **template** — шаблон Pod'а (те же поля, что в `kind: Pod`, но вложены в Deployment).
- **resources** — запрашиваемые (requests) и максимальные (limits) ресурсы. Scheduler использует requests для размещения Pod'ов. Limits ограничивают потребление.

### Применение

```bash
kubectl apply -f deployment.yaml

# Проверяем
kubectl get deployments
kubectl get replicasets
kubectl get pods

# Подробная информация
kubectl describe deployment hello-app
```

Вы увидите 3 Pod'а с именами вида `hello-app-7d8f9b5c6-xxxxx`.

### Самовосстановление

Удалим один Pod:

```bash
kubectl delete pod <имя-pod'а>
kubectl get pods
```

Kubernetes мгновенно создаст новый Pod вместо удалённого. Deployment гарантирует, что всегда работает 3 реплики.

### Масштабирование

```bash
# Изменить количество реплик
kubectl scale deployment hello-app --replicas=5
kubectl get pods   # теперь 5 Pod'ов

# Или через YAML: поменять replicas: 5 и kubectl apply -f deployment.yaml
```

### Rolling Update (обновление без простоя)

Поменяем версию образа:

```bash
kubectl set image deployment/hello-app app=nginx:1.27-alpine
```

Kubernetes выполнит **rolling update**:
1. Создаёт новый Pod с новым образом.
2. Ждёт, пока он станет Ready.
3. Удаляет один старый Pod.
4. Повторяет, пока все Pod'ы не обновятся.

В любой момент работает не менее `replicas - maxUnavailable` Pod'ов (по умолчанию 75%).

Наблюдаем за процессом:

```bash
kubectl rollout status deployment/hello-app
```

### Откат (Rollback)

```bash
# История ревизий
kubectl rollout history deployment/hello-app

# Откатиться к предыдущей версии
kubectl rollout undo deployment/hello-app

# Откатиться к конкретной ревизии
kubectl rollout undo deployment/hello-app --to-revision=1
```

### Стратегии обновления

В `spec.strategy` можно указать:

```yaml
spec:
  strategy:
    type: RollingUpdate          # по умолчанию
    rollingUpdate:
      maxUnavailable: 1          # максимум 1 Pod может быть недоступен
      maxSurge: 1                # максимум 1 дополнительный Pod сверх replicas
```

Альтернатива — `type: Recreate` — убивает все старые Pod'ы, затем создаёт новые. Простой гарантирован, зато обновление быстрее. Используется, когда нельзя запускать две версии одновременно (например, миграция БД).

### Задание

1. Создайте Deployment с 3 репликами nginx (`deployment.yaml`). Убедитесь, что 3 Pod'а запущены.
2. Удалите один Pod вручную. Убедитесь, что Kubernetes создал новый.
3. Масштабируйте до 5 реплик.
4. Выполните rolling update — измените образ на `nginx:1.27-alpine`. Наблюдайте за `kubectl rollout status`.
5. Выполните откат к предыдущей версии.
6. Приложите скриншоты.

---


## Урок 6. Services, Ingress и сетевая модель

### Проблема

Pod'ы эфемерны: при перезапуске Pod получает **новый IP**. Другие сервисы не могут обращаться к Pod'у по IP — он изменится. Нужна стабильная точка доступа.

### Service

**Service** — стабильная сетевая точка доступа к набору Pod'ов. Service находит «свои» Pod'ы по **label selector** и балансирует нагрузку между ними.

Когда вы создаёте Service, Kubernetes:
1. Назначает ему **виртуальный IP** (ClusterIP) — автоматически, вы его не указываете.
2. Регистрирует **DNS-запись**: имя Service → IP. Другие Pod'ы обращаются по имени.

IP случайный и может измениться при пересоздании. Поэтому **никто не использует IP напрямую**. Все обращаются **по имени**.

### Три порта в Service

```yaml
ports:
  - port: 80            # порт самого Service (по нему обращаются клиенты)
    targetPort: 8080    # порт контейнера в Pod'е (на нём слушает ваше приложение)
    nodePort: 30080     # порт на узле (только для NodePort/LoadBalancer)
```

### Типы Service: три уровня доступа

Каждый следующий тип **включает** предыдущий. Это матрёшка, не альтернативы.

#### ClusterIP (по умолчанию) — только внутри кластера

**Конфиг:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-svc
spec:
  type: ClusterIP            # можно не писать — это дефолт
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**Что прописал я:** имя (`db-svc`), порты (`5432 → 5432`).

**Что назначил Kubernetes** (после `kubectl apply`):

```bash
kubectl get service db-svc
```
```
NAME     TYPE        CLUSTER-IP    PORT(S)
db-svc   ClusterIP   10.96.0.20    5432/TCP
```

`10.96.0.20` — Kubernetes выдал сам. Также зарегистрировал DNS: `db-svc → 10.96.0.20`.

**Кто может обратиться:**

```
Из другого Pod'а:    jdbc:postgresql://db-svc:5432/tracker
                                       ^^^^^^ ^^^^
                                       имя из  port из
                                       metadata конфига

С ноутбука:          НИКАК. Адрес 10.96.0.20 не существует за пределами кластера.
```

**Когда использовать:** БД, внутренние сервисы — всё, что не должно быть доступно извне.

#### NodePort — доступ с хоста по IP узла

**Конфиг:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080       # я выбрал, можно не указывать — K8s назначит сам (30000-32767)
```

**Что прописал я:** имя, порты, `nodePort: 30080`.

**Что назначил Kubernetes:**

```bash
kubectl get service app-svc
```
```
NAME      TYPE       CLUSTER-IP    PORT(S)
app-svc   NodePort   10.96.0.15    8080:30080/TCP
```

`10.96.0.15` — назначен автоматически.

Ещё нужен **IP узла** — это IP машины, на которой работает кластер:

```bash
minikube ip
# 192.168.49.2
```

Этот IP — не из конфига Service. Это IP виртуальной машины minikube.

**Кто может обратиться:**

```
Из другого Pod'а:    http://app-svc:8080          ← имя + port
С ноутбука:          http://192.168.49.2:30080     ← minikube ip + nodePort
```

**Когда использовать:** разработка с minikube, демонстрации. В production неудобно — порты 30000-32767, IP узла нужно знать.

#### LoadBalancer — красивый внешний IP от облака

**Конфиг:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      # nodePort не указываю — K8s назначит автоматически
```

**Что прописал я:** имя, `port: 80`, `targetPort: 8080`.

**Что назначил Kubernetes:**

```bash
kubectl get service app-svc
```
```
NAME      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)
app-svc   LoadBalancer   10.96.0.15    54.123.45.67    80:31234/TCP
```

- `10.96.0.15` — внутренний IP, назначен K8s
- `54.123.45.67` — **внешний IP, выдан облачным провайдером** (AWS/GCP). K8s попросил облако: «дай мне внешний IP и направь трафик с порта 80 ко мне». Облако выдало.
- `31234` — nodePort, назначен K8s автоматически (под капотом, пользователь не использует)

**Кто может обратиться:**

```
Из другого Pod'а:         http://app-svc:80            ← имя + port
Из интернета (облако):    http://54.123.45.67:80        ← EXTERNAL-IP + port
```

**В minikube** облака нет, поэтому `EXTERNAL-IP` = `<pending>` навсегда. Используйте:

```bash
minikube service app-svc --url
# http://192.168.49.2:31234     ← minikube подставляет свой IP + автоназначенный nodePort
```

**Когда использовать:** production в облаке. Но каждый LoadBalancer Service = отдельный облачный балансировщик = **деньги** ($15+/мес). Поэтому на практике — один LoadBalancer на Ingress Controller, а не на каждый сервис.

### Откуда что берётся — итоговая таблица

| Значение | Кто задаёт | Откуда |
|----------|-----------|--------|
| `port` | **Я** | Мой конфиг |
| `targetPort` | **Я** | Мой конфиг (порт приложения в контейнере) |
| `nodePort` | **Я или K8s** | Мой конфиг или автоматически (30000-32767) |
| `CLUSTER-IP` | **K8s** | Назначает автоматически |
| `Node IP` | **Машина** | `minikube ip` / IP сервера |
| `EXTERNAL-IP` | **Облако** | Облачный провайдер (AWS/GCP) |
| **DNS-имя** | **Я → K8s** | Я задаю `metadata.name` → K8s создаёт DNS-запись |

### Как сервисы находят друг друга

IP рандомный — поэтому **никто его не использует**. Сервисы обращаются **по имени**. Это ровно как в Docker Compose:

```yaml
# Docker Compose:
services:
  db:                                    # ← имя
    image: postgres:17
  app:
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/tracker
      #                                       ^^  имя сервиса

# Kubernetes Service с metadata.name: db-svc:
      env:
        - name: SPRING_DATASOURCE_URL
          value: "jdbc:postgresql://db-svc:5432/tracker"
          #                         ^^^^^^  имя Service
```

Kubernetes DNS автоматически регистрирует: `db-svc → 10.96.0.20`. IP может быть любым — приложение обращается по имени, DNS разруливает.

### port-forward — для локальной отладки

Быстрый способ получить доступ к любому Service с ноутбука без NodePort/LoadBalancer:

```bash
kubectl port-forward service/db-svc 5432:5432
# Теперь localhost:5432 на вашем ноутбуке → Service db-svc → Pod с PostgreSQL
```

Работает для любого типа Service. Удобно для отладки, но не для production.

### Ingress — как устроен доступ в production

#### Проблема: LoadBalancer на каждый сервис — дорого

10 сервисов × LoadBalancer = 10 внешних IP = 10 × $15 = $150/мес. Плюс: клиент знает 10 адресов, 10 SSL-сертификатов.

#### Решение: один вход, маршрутизация по путям

```
Интернет
    │
    │  https://myapp.com
    ▼
┌──────────────────────────┐
│  LoadBalancer (один!)     │  ← единственный внешний IP
│  54.1.1.1                │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  Ingress Controller      │  ← Pod с nginx внутри кластера
│                          │
│  /api/orders  → order-svc   (ClusterIP)
│  /api/catalog → catalog-svc (ClusterIP)
│  /api/auth    → auth-svc    (ClusterIP)
└──────────────────────────┘
```

Один LoadBalancer ($15/мес) → один Ingress Controller → маршрутизация на множество ClusterIP-сервисов (бесплатно).

#### Установка Ingress Controller

```bash
# В minikube — одна команда
minikube addons enable ingress

# В production — через Helm
helm install ingress-nginx ingress-nginx/ingress-nginx
```

Это создаёт Pod с nginx + один LoadBalancer Service для него.

#### Конфиг Ingress — правила маршрутизации

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: order-svc        # ← имя ClusterIP Service
                port:
                  number: 8080         # ← port из того Service

          - path: /api/catalog
            pathType: Prefix
            backend:
              service:
                name: catalog-svc
                port:
                  number: 8080

          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: auth-svc
                port:
                  number: 8080
```

**Что происходит при запросе:**

```
1. Пользователь: GET https://myapp.com/api/orders/42

2. DNS: myapp.com → 54.1.1.1 (EXTERNAL-IP Ingress Controller'а)

3. nginx (Ingress Controller) смотрит правила:
   path начинается с /api/orders → backend: order-svc:8080

4. nginx отправляет запрос на order-svc:8080
   (K8s DNS: order-svc → 10.96.0.15)

5. Service → Pod → ответ → nginx → пользователь
```

#### Типичная production-схема

```
✅ Один Ingress:

  54.1.1.1 (LB) → nginx (Ingress Controller)
                    ├── /api/orders  → order-svc   (ClusterIP, бесплатно)
                    ├── /api/catalog → catalog-svc  (ClusterIP, бесплатно)
                    └── /api/auth   → auth-svc     (ClusterIP, бесплатно)

                    Итого: $15/мес, 1 IP, 1 сертификат
```

По сути Ingress — это **API Gateway** (из урока по микросервисам), но реализованный средствами Kubernetes.

### Задание

1. Создайте Deployment (3 реплики nginx) и ClusterIP Service.
2. Используйте `kubectl port-forward` для доступа с хоста. Откройте http://localhost:8080.
3. Создайте NodePort Service. Получите URL через `minikube service <n> --url`.
4. Выполните `kubectl get endpoints <service>` — убедитесь, что 3 Pod'а указаны.
5. Включите Ingress в minikube (`minikube addons enable ingress`). Создайте Ingress-ресурс. Проверьте доступ.
6. Приложите скриншоты.

---

## Урок 7. ConfigMaps и Secrets

### Зачем

Конфигурация приложения (URL базы данных, feature flags, таймауты) не должна быть захардкожена в образе. Она должна передаваться извне — через переменные окружения или конфигурационные файлы.

В Docker Compose мы использовали `environment`. В Kubernetes для этого есть **ConfigMap** и **Secret**.

### ConfigMap

ConfigMap — ресурс для хранения неконфиденциальных конфигурационных данных в формате key-value.

**Создание через YAML:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
```

**Создание через kubectl:**

```bash
# Из литералов
kubectl create configmap app-config \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info

# Из файла
kubectl create configmap app-config --from-file=application.properties
```

**Использование в Deployment (как переменные окружения):**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: app-config
      # Или отдельные переменные:
      env:
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
```

**Использование как файл (volume mount):**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

Каждый ключ из ConfigMap станет файлом в `/etc/config/`.

### Secret

Secret — ресурс для хранения конфиденциальных данных (пароли, токены, ключи). Данные хранятся в base64 (не шифрование, а кодирование!).

**Создание через kubectl:**

```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=mysecretpassword
```

**Создание через YAML:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                  # stringData принимает обычный текст (удобнее)
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: mysecretpassword
```

> `stringData` автоматически кодируется в base64 при сохранении. Поле `data` ожидает уже закодированные значения.

**Использование в Deployment:**

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      envFrom:
        - secretRef:
            name: db-credentials
```

### Spring Boot + ConfigMap/Secret

Для Spring Boot всё просто — переменные окружения автоматически переопределяют свойства из `application.properties`:

```yaml
# ConfigMap
data:
  SPRING_DATASOURCE_URL: "jdbc:postgresql://db-svc:5432/tracker"

# Secret
stringData:
  SPRING_DATASOURCE_USERNAME: "postgres"
  SPRING_DATASOURCE_PASSWORD: "password"
```

В Deployment:

```yaml
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: db-credentials
```

### Важные замечания о Secret

- base64 — это **не шифрование**. Любой с доступом к кластеру прочитает секреты.
- Для production используйте **External Secrets Operator** или **Sealed Secrets** для интеграции с HashiCorp Vault, AWS Secrets Manager и т.д.
- Ограничивайте RBAC-доступ к Secret'ам.
- Секреты **не коммитьте** в Git (даже в base64).

### Просмотр и отладка

```bash
kubectl get configmaps
kubectl get secrets

kubectl describe configmap app-config
kubectl describe secret db-credentials

# Декодировать secret
kubectl get secret db-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 --decode
```

### Задание

1. Создайте ConfigMap с настройками приложения.
2. Создайте Secret с учётными данными БД.
3. Создайте Deployment, который использует ConfigMap и Secret как переменные окружения.
4. Зайдите внутрь Pod'а (`kubectl exec -it <pod> -- bash`) и выполните `env | grep SPRING` — убедитесь, что переменные на месте.
5. Приложите скриншоты.

---

## Урок 8. Persistent Volumes

### Проблема

Pod'ы эфемерны — при удалении или перезапуске все данные внутри контейнера теряются. Для баз данных и любых stateful-сервисов нужно **постоянное хранилище**.

В Docker мы решали это **томами** (`-v` / `volumes`). В Kubernetes механизм сложнее, потому что Pod может оказаться на **любом** узле кластера.

### Трёхуровневая модель

```
┌──────────────────────────────────────┐
│         PersistentVolumeClaim (PVC)  │  ← Запрос: "мне нужен 1Gi диск"
│         (создаёт разработчик)        │
└───────────────┬──────────────────────┘
                │ привязка (bound)
┌───────────────┴──────────────────────┐
│         PersistentVolume (PV)        │  ← Реальный ресурс хранилища
│         (создаёт админ или           │
│          динамически StorageClass)   │
└───────────────┬──────────────────────┘
                │
┌───────────────┴──────────────────────┐
│         Физическое хранилище         │  ← NFS, EBS, SSD, hostPath...
└──────────────────────────────────────┘
```

- **PersistentVolume (PV)** — описание реального хранилища (диск, NFS-шара и т.д.).
- **PersistentVolumeClaim (PVC)** — запрос от приложения: «мне нужен диск на X гигабайт».
- **StorageClass** — шаблон для **динамического** создания PV (в облаках создаёт диск автоматически).

### Пример: PostgreSQL с персистентным хранилищем

**1. PersistentVolumeClaim (`pvc.yaml`):**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce        # один узел может писать
  resources:
    requests:
      storage: 1Gi
```

В minikube динамическое создание PV работает из коробки (StorageClass `standard`).

**2. Deployment PostgreSQL (`postgres-deployment.yaml`):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: db-credentials
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
```

**3. Service для PostgreSQL (`postgres-service.yaml`):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### Применение

```bash
# Secret (из прошлого урока)
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=password \
  --from-literal=POSTGRES_DB=tracker

kubectl apply -f pvc.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml

# Проверяем
kubectl get pvc
kubectl get pv
kubectl get pods
```

### Проверка персистентности

```bash
# Подключаемся к PostgreSQL
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U postgres -d tracker

# Создаём таблицу
CREATE TABLE items(id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO items(name) VALUES ('test');
SELECT * FROM items;
\q

# Удаляем Pod (Deployment создаст новый)
kubectl delete pod -l app=postgres

# Ждём нового Pod'а
kubectl get pods -w

# Подключаемся к НОВОМУ Pod'у — данные на месте!
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- \
  psql -U postgres -d tracker -c "SELECT * FROM items;"
```

### Access Modes

| Mode | Сокращение | Описание |
|------|-----------|---------|
| ReadWriteOnce | RWO | Один узел может монтировать для чтения и записи |
| ReadOnlyMany | ROX | Много узлов могут монтировать для чтения |
| ReadWriteMany | RWX | Много узлов могут монтировать для чтения и записи |

Для баз данных обычно используется **RWO** — один Pod пишет, один диск.

### StatefulSet (кратко)

Для stateful-приложений (БД, Kafka, Elasticsearch) вместо Deployment лучше использовать **StatefulSet**. Он гарантирует:
- Стабильные имена Pod'ов (`postgres-0`, `postgres-1`).
- Стабильные тома (каждый Pod получает свой PVC).
- Порядок запуска и остановки.

Мы не будем подробно изучать StatefulSet в этом курсе, но знайте о его существовании.

### Задание

1. Создайте PVC, Deployment PostgreSQL с томом и Service.
2. Создайте базу данных и таблицу.
3. Удалите Pod PostgreSQL. Убедитесь, что новый Pod видит данные.
4. Приложите скриншоты.

---

## Урок 9. Helm

### Проблема

Для одного приложения нужно создать 5–10 YAML-файлов: Deployment, Service, ConfigMap, Secret, PVC, Ingress... Для разных сред (dev, staging, prod) настройки отличаются. Копировать и редактировать YAML вручную — рутинно и чревато ошибками.

### Что такое Helm

**Helm** — пакетный менеджер для Kubernetes. Аналогия:
- `apt` / `brew` — для ОС
- `maven` — для Java-зависимостей
- **`helm`** — для Kubernetes-приложений

Helm оперирует **чартами** (charts) — пакетами, содержащими шаблоны Kubernetes-манифестов + значения по умолчанию.

### Установка

**🪟 Windows:**

```powershell
choco install kubernetes-helm
```

**🍎 macOS:**

```bash
brew install helm
```

**🐧 Linux:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Проверка:**

```bash
helm version
```

### Helm: основные концепции

- **Chart** — набор файлов, описывающих Kubernetes-ресурсы. Аналог «пакета».
- **Release** — конкретная инсталляция чарта в кластер. Один чарт можно установить несколько раз под разными именами.
- **Repository** — хранилище чартов (аналог Docker Hub для образов).
- **Values** — параметры, которые подставляются в шаблоны.

### Использование готовых чартов

Helm имеет огромную экосистему готовых чартов. Пример — установка PostgreSQL:

```bash
# Добавляем репозиторий Bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Ищем чарт
helm search repo postgresql

# Устанавливаем PostgreSQL
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=password \
  --set auth.database=tracker \
  --set primary.persistence.size=1Gi

# Проверяем
kubectl get pods
kubectl get services
helm list
```

Одной командой развёрнут PostgreSQL с PVC, Service, Secret, readiness/liveness probes и прочими best practices.

### Управление релизами

```bash
# Список установленных релизов
helm list

# Информация о релизе
helm status my-postgres

# Обновление (изменение параметров)
helm upgrade my-postgres bitnami/postgresql \
  --set auth.postgresPassword=newpassword

# Откат
helm rollback my-postgres 1

# Удаление
helm uninstall my-postgres
```

### Создание своего чарта

```bash
helm create my-app
```

Создаётся структура:

```
my-app/
├── Chart.yaml          # метаданные чарта (имя, версия)
├── values.yaml         # значения по умолчанию
├── templates/          # шаблоны Kubernetes-манифестов
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # вспомогательные шаблоны
│   └── ...
└── charts/             # зависимости (другие чарты)
```

### Шаблонизация

Шаблоны Helm используют Go template syntax:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.targetPort }}
```

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry/myapp
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080
```

### Установка своего чарта

```bash
# Превью — посмотреть, что получится (без установки)
helm template my-release ./my-app

# Dry-run — проверить валидность
helm install my-release ./my-app --dry-run

# Установка
helm install my-release ./my-app

# Установка с переопределением значений
helm install my-release ./my-app \
  --set replicaCount=5 \
  --set image.tag=2.0.0

# Или через файл значений
helm install my-release ./my-app -f production-values.yaml
```

### Файлы значений для разных сред

```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "latest"

# values-prod.yaml
replicaCount: 5
image:
  tag: "1.2.3"
```

```bash
helm install app ./my-app -f values-dev.yaml     # dev
helm install app ./my-app -f values-prod.yaml     # prod
```

### Задание

1. Установите Helm.
2. Установите PostgreSQL из чарта Bitnami. Подключитесь к нему и создайте таблицу.
3. Создайте свой чарт (`helm create`) для nginx. Настройте `values.yaml` (replicas, image, порты).
4. Установите свой чарт в кластер.
5. Приложите скриншоты.

---

## Урок 10. Деплой Spring Boot приложения в Kubernetes

В этом уроке мы соберём всё вместе: задеплоим Spring Boot приложение с PostgreSQL в Kubernetes-кластер.

### Что мы создадим

```
                        ┌───── Kubernetes Cluster ─────┐
                        │                              │
Internet ──▶ Ingress ──▶│  Service (app) ──▶ Deployment│
                        │       │              (3 Pod)  │
                        │       │                       │
                        │  Service (db) ──▶ Deployment  │
                        │                   (1 Pod)     │
                        │                      │        │
                        │                    PVC        │
                        │                      │        │
                        │                    PV         │
                        └───────────────────────────────┘
```

### Шаг 1: Загрузка образа в minikube

minikube не имеет доступа к вашему локальному Docker registry. Есть несколько способов загрузить образ:

**Способ A: Использовать Docker daemon minikube**

```bash
# Переключаем Docker-клиент на Docker внутри minikube
eval $(minikube docker-env)

# Теперь docker build собирает образ ВНУТРИ minikube
docker build -t job4j/tracker:1.0 .

# Вернуться к локальному Docker
eval $(minikube docker-env -u)
```

**Способ B: minikube image load**

```bash
# Собираем локально
docker build -t job4j/tracker:1.0 .

# Загружаем в minikube
minikube image load job4j/tracker:1.0
```

> **Важно**: при использовании локального образа в манифесте укажите `imagePullPolicy: Never` или `IfNotPresent`, иначе Kubernetes попытается скачать его из Docker Hub.

### Шаг 2: Secret для БД

```bash
kubectl create secret generic tracker-db-secret \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=password \
  --from-literal=POSTGRES_DB=tracker
```

### Шаг 3: ConfigMap для приложения

Файл `k8s/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tracker-config
data:
  SPRING_DATASOURCE_URL: "jdbc:postgresql://tracker-db:5432/tracker"
  SPRING_DATASOURCE_DRIVER_CLASS_NAME: "org.postgresql.Driver"
```

### Шаг 4: PostgreSQL

Файл `k8s/postgres.yaml`:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tracker-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tracker-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tracker-db
  template:
    metadata:
      labels:
        app: tracker-db
    spec:
      containers:
        - name: postgres
          image: postgres:17
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: tracker-db-secret
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: db-storage
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: tracker-db-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: tracker-db
spec:
  type: ClusterIP
  selector:
    app: tracker-db
  ports:
    - port: 5432
      targetPort: 5432
```

### Шаг 5: Приложение

Файл `k8s/app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tracker-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tracker-app
  template:
    metadata:
      labels:
        app: tracker-app
    spec:
      containers:
        - name: app
          image: job4j/tracker:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: tracker-config
            - secretRef:
                name: tracker-db-secret
          env:
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: tracker-db-secret
                  key: POSTGRES_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: tracker-db-secret
                  key: POSTGRES_PASSWORD
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: tracker-app
spec:
  type: NodePort
  selector:
    app: tracker-app
  ports:
    - port: 8080
      targetPort: 8080
```

### Шаг 6: Деплой

```bash
# Создаём namespace (опционально, но хорошая практика)
kubectl create namespace tracker
kubectl config set-context --current --namespace=tracker

# Или деплоим в default namespace:
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/app.yaml

# Или всё сразу (если файлы в одной папке):
kubectl apply -f k8s/

# Следим за запуском
kubectl get pods -w
```

### Шаг 7: Проверка

```bash
# Все ресурсы
kubectl get all

# Статус Pod'ов
kubectl get pods

# Логи приложения
kubectl logs -f deployment/tracker-app

# Логи по label (удобно, когда Pod'ов несколько)
kubectl logs -l app=tracker-app --all-containers

# Доступ через minikube
minikube service tracker-app --url

# Или через port-forward
kubectl port-forward service/tracker-app 8080:8080
```

После получения URL (через `minikube service` или `port-forward`) проверяем, что приложение отвечает:

```bash
# Health check
curl http://localhost:8080/actuator/health
# {"status":"UP"}

# Или через URL от minikube (пример):
curl http://192.168.49.2:30080/actuator/health
```

Если приложение использует авторизацию (JWT), проверяйте через Postman или curl с токеном:

```bash
# Получаем токен
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password"}'

# Используем токен
curl http://localhost:8080/api/orders \
  -H "Authorization: Bearer <token>"
```

> **Совет**: команда `minikube service <name>` автоматически открывает URL в браузере. Если браузер не открылся — используйте флаг `--url` и перейдите по выданному адресу вручную.

### Liveness и Readiness Probes

В манифесте приложения мы указали два типа проверок:

- **readinessProbe** — Pod готов принимать трафик? Если нет, Service не направляет к нему запросы. Используется при запуске (Spring Boot стартует 20–30 секунд) и при деградации.
- **livenessProbe** — Pod жив? Если нет, Kubernetes **перезапускает** контейнер.

```yaml
readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 30    # ждём 30 сек перед первой проверкой
  periodSeconds: 10           # проверяем каждые 10 сек

livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60    # ждём дольше — даём приложению стартовать
  periodSeconds: 30
```

> Для Spring Boot добавьте зависимость `spring-boot-starter-actuator`.

### Обновление приложения

```bash
# Собираем новую версию
docker build -t job4j/tracker:2.0 .
minikube image load job4j/tracker:2.0

# Обновляем образ
kubectl set image deployment/tracker-app app=job4j/tracker:2.0

# Наблюдаем за rolling update
kubectl rollout status deployment/tracker-app
```

### Итоговая структура проекта

```
job4j_tracker/
├── src/
├── pom.xml
├── Dockerfile
└── k8s/
    ├── configmap.yaml
    ├── postgres.yaml       # PVC + Deployment + Service
    └── app.yaml            # Deployment + Service
```

### Чек-лист Kubernetes-деплоя

- [ ] Dockerfile использует multi-stage build
- [ ] Образ загружен в minikube (`minikube image load`)
- [ ] Secret создан для паролей
- [ ] ConfigMap создан для конфигурации
- [ ] Deployment с readiness/liveness probes
- [ ] Resources (requests/limits) указаны
- [ ] PVC для базы данных
- [ ] Service для каждого компонента
- [ ] Labels согласованы между Deployment и Service

### «Стопка файлов» — Docker Compose vs Kubernetes

Законный вопрос: в Docker Compose один файл на 30 строк, а в Kubernetes — 8 файлов на 200 строк. Зачем?

**Docker Compose** решает одну задачу: «подними контейнеры на одной машине». Один файл — достаточно.

**Kubernetes** решает десять задач: запуск, масштабирование, отказоустойчивость, сеть, хранение, конфигурация, секреты, обновления, мониторинг, маршрутизация. Каждая задача — отдельный ресурс — отдельный (или объединённый через `---`) YAML.

Но в реальности **никто не пишет все YAML руками каждый раз**:

| Этап | Как работают с YAML |
|------|-------------------|
| **Учёба** | Пишем руками, 5–8 файлов. Это нормально — нужно понять, что делает каждый ресурс. |
| **Средний проект** | **Helm** — шаблонизатор. Один раз написал шаблон, дальше `helm install my-app ./chart --set replicas=3` генерирует все файлы автоматически. |
| **Большая компания** | Вообще не пишешь K8s-манифесты. Платформенная команда дала шаблон. Ты заполняешь `values.yaml` на 15 строк. CI/CD генерирует всё остальное. |

Наш проект в Helm-формате выглядел бы так:

```bash
# Вместо 8 kubectl apply:
helm install tracker ./tracker-chart \
  --set image.tag=1.0 \
  --set replicas=3 \
  --set db.password=secret
```

Одна команда — и все Deployment, Service, ConfigMap, Secret, PVC, Ingress созданы. Helm мы разобрали в уроке 9.

### Задание

1. Создайте Dockerfile для вашего Spring Boot проекта (job4j_tracker или CheckDev).
2. Загрузите образ в minikube.
3. Создайте все необходимые Kubernetes-манифесты (Secret, ConfigMap, PVC, Deployments, Services).
4. Задеплойте приложение и БД в кластер.
5. Убедитесь:
   - Приложение доступно через `minikube service` или `port-forward`.
   - После удаления Pod'а PostgreSQL данные сохраняются.
   - После удаления Pod'а приложения он автоматически перезапускается.
6. Выполните rolling update — обновите версию образа приложения.
7. Оставьте ссылку на коммит. В проекте должна быть папка `k8s/` со всеми манифестами.
8. Добавьте в README инструкцию по деплою в Kubernetes.
