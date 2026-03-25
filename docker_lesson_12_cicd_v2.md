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
└── .env
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

  agent1:
    build:
      context: ./agent
      args:
        DOCKER_GID: ${DOCKER_GID}
    container_name: agent1
    environment:
      - JENKINS_AGENT_SSH_PUBKEY=${JENKINS_SSH_PUBKEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - jenkins-net
    restart: unless-stopped

volumes:
  jenkins_home:

networks:
  jenkins-net:
```

**agent/Dockerfile:**

```dockerfile
FROM jenkins/ssh-agent:jdk21

# Устанавливаем Docker CLI, чтобы агент мог вызывать docker build
USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends docker.io && \
    rm -rf /var/lib/apt/lists/*

# Добавляем пользователя jenkins в группу docker.
# GID берём с хост-машины, чтобы совпали права на docker.sock.
ARG DOCKER_GID
RUN groupadd -g ${DOCKER_GID} dockerhost || true && \
    usermod -aG ${DOCKER_GID} jenkins

USER jenkins
```

> **Зачем отдельный Dockerfile для агента?** Базовый образ `jenkins/ssh-agent:jdk21` не содержит Docker CLI. Нам нужно его доустановить и настроить права. Это как раз задача для Dockerfile — воспроизводимо, версионируемо, без ручных команд.

**.env:**

```bash
# GID группы docker на хост-машине.
# Узнать: getent group docker | cut -d: -f3
DOCKER_GID=999

# SSH-публичный ключ для подключения Jenkins к агенту.
# Генерируется в Jenkins: Manage Jenkins → Credentials → SSH key
JENKINS_SSH_PUBKEY=ssh-rsa AAAA...
```

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

### Dockerfile проекта — один на все случаи

Частый вопрос: «нужен ли отдельный Dockerfile для CI?» Нет. Dockerfile один, лежит в корне проекта — рядом с `build.gradle.kts` или `pom.xml`:

```
job4j_devops/
├── src/
├── build.gradle.kts
├── settings.gradle.kts
├── Dockerfile            ← вот он
├── compose.yaml
└── .dockerignore
```

Это тот же Dockerfile из Урока 9 — с multi-stage сборкой:

```dockerfile
FROM gradle:8.12-jdk21 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts gradle.properties ./
COPY gradle/libs.versions.toml ./gradle/
RUN gradle --no-daemon dependencies
COPY src ./src
RUN gradle --no-daemon clean build -x test

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Когда Jenkins вызывает `docker build`, Docker сам выполняет всю сборку внутри контейнера — скачивает зависимости, компилирует, собирает JAR, и упаковывает в минимальный образ. Jenkins'у не нужно знать ничего про Gradle или Java.

> **А если Jenkins уже собрал JAR на предыдущем этапе?** Это нормально — Docker пересоберёт в своём контейнере. Да, двойная работа. Но зато Dockerfile самодостаточный: он работает и в CI, и у разработчика на ноутбуке, и в любом другом месте. Один Dockerfile — один результат. Это ценнее, чем экономия минуты на сборке.

### Jenkinsfile

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

Обратите внимание: Jenkins **не вызывает** `gradle build` отдельно. Dockerfile всё делает сам. Пайплайн максимально простой: checkout → docker build.

Тег `${BUILD_NUMBER}` — номер сборки Jenkins. Каждый образ получает уникальный тег: `job4j_devops:1`, `job4j_devops:2`, `job4j_devops:3`... Это позволяет откатиться на любую версию.

### Пуш образа в Registry

Собранный образ живёт только на хосте. Чтобы его использовать на production-сервере — нужно залить в registry:

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

Пароль хранится в Jenkins Credentials, не в Jenkinsfile. `--password-stdin` передаёт пароль через pipe — он не попадёт в логи или `ps aux`.

### Полный пайплайн

```
Developer pushes to Git
        │
        ▼
Jenkins detects change (webhook)
        │
        ▼
┌─ Pipeline ─────────────────────┐
│  1. Checkout                   │
│  2. Docker Build               │  ← Dockerfile делает всю сборку
│  3. Docker Push to Registry    │
└────────────────────────────────┘
        │
        ▼
Production pulls new image
```

> **Из практики**: всегда тегируйте образы номером сборки или хешем коммита. Тег `latest` — удобный указатель на «самое свежее», но если что-то сломалось, вы должны мочь откатиться на конкретную версию: `docker pull myuser/job4j_devops:42`.

### Задание

1. Создайте директорию `jenkins-infra` с `compose.yaml` и `agent/Dockerfile` по примеру из урока.
2. Запустите Jenkins через `docker compose up -d`.
3. Настройте агент в Jenkins (Manage Jenkins → Nodes → New Node).
4. В проекте job4j_devops добавьте Jenkinsfile с этапом Docker Build.
5. Запустите сборку. Приложите скриншот успешного этапа Docker Build.
