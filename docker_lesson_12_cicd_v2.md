# Урок 12. Docker в CI/CD (Jenkins)

В этом уроке мы добавим в Jenkins этап сборки Docker-образа. Всю инфраструктуру (Jenkins + агент с Docker) опишем через Compose.

### Зачем Docker в CI/CD

До этого мы собирали Docker-образы вручную на своей машине. В реальном проекте так не делают — образы собираются автоматически при каждом пуше в Git:

```
Push в Git → Jenkins обнаруживает изменения → Тесты → Сборка JAR → Сборка Docker-образа → Push в Registry
```

### Инфраструктура через Compose

Раз у нас курс по Docker — опишем Jenkins и его агент через `compose.yaml`. Никакой ручной настройки.

**Структура:**

```
jenkins-infra/
├── compose.yaml
├── agent/
│   └── Dockerfile
├── .env              ← не коммитить! добавьте в .gitignore
├── .env.example      ← шаблон с пустыми значениями (коммитить)
└── .gitignore
```

**compose.yaml:**

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk21
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
    networks:
      - jenkins-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/login"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  agent1:
    build: ./agent
    container_name: agent1
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=${JENKINS_SSH_PUBKEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    group_add:
      - ${DOCKER_GID}
    depends_on:
      jenkins:
        condition: service_healthy
    networks:
      - jenkins-net
    restart: unless-stopped

volumes:
  jenkins_home:

networks:
  jenkins-net:
```

`group_add` — добавляет GID группы docker с хоста к процессу контейнера на уровне runtime. Это аналог `--group-add` в `docker run`. Никаких `groupadd`/`usermod` внутри Dockerfile не нужно — Compose делает всё сам.

**agent/Dockerfile:**

```dockerfile
FROM jenkins/ssh-agent:jdk21

USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg \
      -o /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/debian bookworm stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*
```

Устанавливаем `docker-ce-cli` из официального репозитория Docker — это только CLI, без daemon'а. Пакет `docker.io` из Debian-репозитория в актуальных версиях не содержит CLI-бинарник, поэтому использовать его не стоит.

**.env:**

```bash
# GID группы docker на хост-машине.
# Узнать: getent group docker | cut -d: -f3
DOCKER_GID=999

# SSH-публичный ключ для подключения Jenkins к агенту.
# Генерируется в Jenkins: Manage Jenkins → Credentials → SSH key
JENKINS_SSH_PUBKEY=ssh-rsa AAAA...
```

> **Напоминание из Урока 10**: `.env` содержит чувствительные данные — **не коммитьте** его в Git. Добавьте в `.gitignore`. Вместо этого создайте файл `.env.example` с пустыми значениями — он покажет коллегам, какие переменные нужно заполнить.

**Запуск:**

```bash
cd jenkins-infra
docker compose up -d
```

Всё. Jenkins доступен на `http://localhost:8080`, агент подключается автоматически.

### Как это работает: Docker socket

Обратите внимание на строку в `compose.yaml`:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Docker socket (`/var/run/docker.sock`) — это Unix-сокет, через который Docker CLI общается с Docker Daemon. Пробросив его в контейнер агента, мы даём агенту возможность управлять Docker'ом на хосте.

Но пробросить файл мало — процессу внутри контейнера нужны права на чтение/запись в этот сокет. На хосте сокет принадлежит группе `docker` с определённым GID (обычно 999). Именно поэтому в `compose.yaml` мы указываем `group_add: - ${DOCKER_GID}` — это добавляет GID группы docker с хоста к процессу jenkins внутри контейнера.

```
┌──────────────────────────────────────────┐
│  Host Machine                            │
│                                          │
│  Docker Daemon (dockerd)                 │
│       ▲                                  │
│       │ /var/run/docker.sock             │
│       │                                  │
│  ┌────┴──────────┐  ┌────────────────┐   │
│  │  agent1       │  │  jenkins       │   │
│  │  docker CLI ──┘  │  (controller)  │   │
│  │  + JDK 21       │                │   │
│  └───────────────┘  └────────────────┘   │
│       jenkins-net                        │
└──────────────────────────────────────────┘
```

Это **не** Docker-in-Docker. Здесь один Docker Daemon — на хосте. Агент просто умеет отправлять ему команды. Когда агент выполняет `docker build`, образ появляется на хосте.

> **Важно про безопасность**: доступ к Docker socket = root-доступ к хосту. Контейнер с socket'ом может запустить `docker run -v /:/host ubuntu rm -rf /host`. Пробрасывайте socket только доверенным контейнерам.

### Dockerfile проекта: два подхода

Есть два подхода, оба валидны. Выбор зависит от того, что делает ваш пайплайн.

**Подход 1: Один Dockerfile (самодостаточный)**

Dockerfile содержит multi-stage сборку. Jenkins просто вызывает `docker build` — вся сборка происходит внутри Docker:

```
job4j_devops/
├── src/
├── build.gradle.kts
├── Dockerfile            ← полный, с этапом сборки
├── compose.yaml
└── .dockerignore
```

```dockerfile
FROM gradle:8.12-jdk21 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts gradle.properties ./
COPY gradle/libs.versions.toml ./gradle/
RUN gradle --no-daemon dependencies
COPY src ./src
RUN gradle --no-daemon clean build

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Плюс: Dockerfile работает везде одинаково — на ноутбуке, в CI, на любом сервере. Один файл — один результат. Тесты гарантированно прогоняются при каждой сборке.
Минус: если Jenkins уже собрал JAR и прогнал тесты, Docker повторит эту работу заново.

**Подход 2: Два Dockerfile (CI упаковывает готовый артефакт)**

Jenkins собирает JAR, прогоняет тесты. Отдельный Dockerfile только упаковывает готовый артефакт:

```
job4j_devops/
├── src/
├── build.gradle.kts
├── Dockerfile            ← для разработчика (полный, с multi-stage)
├── jenkins/
│   └── Dockerfile        ← для CI (только упаковка)
├── compose.yaml
└── .dockerignore
```

`jenkins/Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

В Jenkinsfile:

```groovy
sh 'docker build -f jenkins/Dockerfile -t job4j_devops:${BUILD_NUMBER} .'
```

Плюс: CI не тратит время на повторную сборку, пайплайн быстрее.
Минус: два Dockerfile нужно поддерживать. Обновил базовый образ в одном — не забудь обновить в другом.

> **Рекомендация**: для учебных проектов используйте подход 1 — проще. Для production-проектов с развитым CI — подход 2 экономит минуты на каждой сборке.

### Jenkinsfile

**Подход 1** — самодостаточный Dockerfile, Jenkins только запускает `docker build`:

```groovy
pipeline {
    agent { label 'agent1' }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your/job4j_devops.git'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t job4j_devops:${BUILD_NUMBER} .'
            }
        }
    }

    post {
        always {
            script {
                def info = """
                    Build: ${currentBuild.number}
                    Status: ${currentBuild.currentResult}
                    Duration: ${currentBuild.durationString}
                """
                echo info
            }
        }
    }
}
```

Тесты здесь запускаются **внутри** `docker build` (в этапе `RUN gradle clean build`). Если тесты упадут — сборка образа тоже упадёт.

**Подход 2** — Jenkins тестирует и собирает JAR, Docker только упаковывает:

```groovy
pipeline {
    agent { label 'agent1' }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/your/job4j_devops.git'
            }
        }
        stage('Test') {
            steps {
                sh 'chmod +x ./gradlew'
                sh './gradlew check'
            }
        }
        stage('Build JAR') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -f jenkins/Dockerfile -t job4j_devops:${BUILD_NUMBER} .'
            }
        }
    }

    post {
        always {
            script {
                def info = """
                    Build: ${currentBuild.number}
                    Status: ${currentBuild.currentResult}
                    Duration: ${currentBuild.durationString}
                """
                echo info
            }
        }
    }
}
```

Обратите внимание на `-f jenkins/Dockerfile` — указываем путь к CI-версии Dockerfile.

Тег `${BUILD_NUMBER}` — номер сборки Jenkins. Каждый образ получает уникальный тег: `job4j_devops:1`, `job4j_devops:2`, `job4j_devops:3`... Это позволяет откатиться на любую версию.

### Пуш образа в Registry

Собранный образ живёт только на хосте. Чтобы его использовать на production-сервере — нужно залить в registry. Разберём по шагам.

#### Шаг 1: Регистрация на Docker Hub

Docker Hub — публичный registry по умолчанию. Бесплатный план позволяет хранить неограниченное количество публичных образов и один приватный.

1. Перейдите на https://hub.docker.com и нажмите **Sign Up**.
2. Заполните: username, email, пароль.
3. Подтвердите email.

Ваш username — это namespace для образов. Если username = `petrov`, образы будут называться `petrov/job4j_devops:latest`.

> **Другие registry**: Docker Hub — не единственный вариант. GitHub Container Registry (`ghcr.io`), GitLab Registry, Yandex Container Registry — всё это работает по тому же принципу. Мы используем Docker Hub как самый простой для начала.

#### Шаг 2: Создание Access Token

Не используйте пароль от аккаунта для CI. Создайте отдельный токен с ограниченными правами:

1. Docker Hub → **Account Settings** → **Personal access tokens** → **Generate new token**
2. Description: `jenkins-ci` (чтобы потом понимать, для чего токен)
3. Access permissions: **Read & Write** (достаточно для push/pull)
4. Нажмите **Generate**, скопируйте токен — он покажется только один раз

> **Почему токен, а не пароль?** Токен можно отозвать, не меняя пароль от аккаунта. Если Jenkins скомпрометирован — отзываете токен, создаёте новый. Аккаунт остаётся в безопасности.

#### Шаг 3: Проверка с локальной машины

Прежде чем настраивать Jenkins, убедитесь, что всё работает:

```bash
# Логин (вместо пароля — токен)
docker login -u <ваш_username>
# Password: <вставляете токен>

# Тегируем образ (формат: username/имя:тег)
docker tag job4j_devops:latest <ваш_username>/job4j_devops:latest

# Пушим
docker push <ваш_username>/job4j_devops:latest

# Проверяем — образ появился на Docker Hub
# https://hub.docker.com/r/<ваш_username>/job4j_devops
#
# Репозиторий создавать вручную НЕ нужно —
# Docker Hub создаст его автоматически при первом push (публичный).
# Если нужен приватный — тогда сначала создайте через UI:
# Docker Hub → Repositories → Create Repository → Private

# Выходим
docker logout
```

#### Шаг 4: Сохранение учётных данных в Jenkins

Учётные данные хранятся в Jenkins Credentials — зашифрованном хранилище. Никогда не пишите токены в Jenkinsfile или `.env`.

1. Jenkins → **Manage Jenkins** → **Credentials**
2. Выберите домен (обычно **Global**) → **Add Credentials**
3. Заполните:
   - **Kind**: Username with password
   - **Username**: ваш Docker Hub username
   - **Password**: токен (не пароль от аккаунта!)
   - **ID**: `dockerhub-creds` (это ID, по которому Jenkinsfile будет обращаться к учётным данным)
   - **Description**: `Docker Hub Access Token`
4. Нажмите **Create**

#### Шаг 5: Этап push в Jenkinsfile

```groovy
stage('Docker Push') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-creds',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {
            sh '''
                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                docker tag job4j_devops:${BUILD_NUMBER} $DOCKER_USER/job4j_devops:${BUILD_NUMBER}
                docker tag job4j_devops:${BUILD_NUMBER} $DOCKER_USER/job4j_devops:latest
                docker push $DOCKER_USER/job4j_devops:${BUILD_NUMBER}
                docker push $DOCKER_USER/job4j_devops:latest
                docker logout
            '''
        }
    }
}
```

Разберём:

- `withCredentials` — Jenkins подставляет логин и токен в переменные окружения `DOCKER_USER` и `DOCKER_PASS`. Они существуют только внутри этого блока.
- `credentialsId: 'dockerhub-creds'` — тот самый ID из шага 4.
- `echo $DOCKER_PASS | docker login --password-stdin` — передаёт токен через pipe. Если написать `docker login -u user -p password`, пароль попадёт в `ps aux` и в логи Jenkins. `--password-stdin` это предотвращает.
- `docker tag` — Docker Hub требует формат `username/image:tag`. Локальный образ `job4j_devops:42` переименовывается в `petrov/job4j_devops:42`.
- Два тега: `${BUILD_NUMBER}` (для отката на конкретную версию) и `latest` (удобный указатель на последнюю).
- `docker logout` — удаляет сохранённый токен с агента после пуша.

### Полный пайплайн

```
Developer pushes to Git
        │
        ▼
Jenkins detects change (webhook)
        │
        ▼
┌─ Подход 1 ──────────────────────┐  ┌─ Подход 2 ──────────────────────┐
│  1. Checkout                    │  │  1. Checkout                    │
│  2. Docker Build                │  │  2. Test (gradlew check)        │
│     (тесты + сборка внутри)     │  │  3. Build JAR (gradlew build)   │
│  3. Docker Push to Registry     │  │  4. Docker Build (-f jenkins/)  │
└─────────────────────────────────┘  │  5. Docker Push to Registry     │
                                     └─────────────────────────────────┘
        │
        ▼
Production pulls new image
```

> **Из практики**: всегда тегируйте образы номером сборки или хешем коммита. Тег `latest` — удобный указатель на «самое свежее», но если что-то сломалось, вы должны мочь откатиться на конкретную версию: `docker pull myuser/job4j_devops:42`.

### Задание

1. Зарегистрируйтесь на Docker Hub, если ещё нет аккаунта.
2. Создайте Access Token с правами Read & Write.
3. С локальной машины: соберите образ, залогиньтесь, запушьте образ на Docker Hub. Убедитесь, что он появился в вашем аккаунте. Приложите скриншот страницы образа на Docker Hub.
4. Создайте директорию `jenkins-infra` с `compose.yaml` и `agent/Dockerfile` по примеру из урока. Запустите Jenkins через `docker compose up -d`.
5. Добавьте Docker Hub credentials в Jenkins (Manage Jenkins → Credentials).
6. В проекте job4j_devops добавьте Jenkinsfile с этапами Docker Build и Docker Push.
7. Запустите сборку. Приложите скриншот успешного пайплайна и ссылку на образ в Docker Hub.
