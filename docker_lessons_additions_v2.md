# Дополнения к Docker-урокам v2

> Ниже — три блока контента, которого не хватает в `docker_lessons_v2.md`.
> Рекомендуемый порядок: вставить после текущего Урока 9 (Multi-stage Build).

---

## Урок 9 — дополнение: Gradle-проекты и продвинутый jlink

> Этот раздел добавляется **в конец** текущего Урока 9 (Многоэтапная сборка), перед Уроком 10.

### Multi-stage для Gradle

До этого мы рассматривали Maven-проекты. Но многие проекты (включая job4j_devops) используют Gradle. Принцип тот же — кэшируем зависимости отдельно от исходников:

```dockerfile
# ── Этап 1: сборка ─────────────────────────────
FROM gradle:8.12-jdk21 AS build
WORKDIR /app

# Сначала — только файлы описания проекта.
# Если они не менялись, Docker возьмёт зависимости из кэша.
COPY build.gradle.kts settings.gradle.kts gradle.properties ./

# Для Gradle Wrapper — копируем ещё и wrapper
# COPY gradlew ./
# COPY gradle/wrapper ./gradle/wrapper

RUN gradle --no-daemon dependencies

# Теперь — исходники. Этот слой пересобирается при каждом изменении кода.
COPY src ./src
RUN gradle --no-daemon clean build -x test

# ── Этап 2: запуск ─────────────────────────────
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Почему `--no-daemon`?** Gradle Daemon полезен на машине разработчика — он живёт между запусками и ускоряет повторные сборки. Но в Docker каждый `RUN` — это чистый слой, между ними нет «живого» процесса. Daemon стартует, собирает, и тут же умирает вместе со слоем. Это бесполезные накладные расходы на старт JVM. `--no-daemon` убирает их.

#### Gradle Wrapper vs образ `gradle:`

В примере выше мы используем официальный образ `gradle:8.12-jdk21`. Но если в проекте есть `gradlew` (Gradle Wrapper) — лучше использовать его. Wrapper гарантирует, что будет именно та версия Gradle, которую ожидает проект:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY gradlew ./
COPY gradle/wrapper ./gradle/wrapper
RUN chmod +x gradlew

COPY build.gradle.kts settings.gradle.kts gradle.properties ./
RUN ./gradlew --no-daemon dependencies

COPY src ./src
RUN ./gradlew --no-daemon clean build -x test

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Из практики**: если вы контролируете образ и точно знаете версию Gradle — `gradle:8.12-jdk21` удобнее (не нужно копировать wrapper-файлы). Если это open-source проект или у вас в команде строго фиксирована версия через wrapper — используйте `gradlew`.

### jdeps + jlink: минимальный JRE

В Уроке 9 мы упомянули jlink. Теперь разберём подробно — это мощный инструмент, который может уменьшить образ в 3-5 раз.

**Идея**: полный JRE содержит ~70 модулей (GUI, CORBA, XML-трансформации...), но вашему Spring Boot приложению нужны далеко не все. `jdeps` анализирует байткод и определяет, какие модули реально используются. `jlink` собирает кастомный JRE только из этих модулей.

#### Шаг за шагом

```dockerfile
FROM gradle:8.12-jdk21 AS build
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts gradle.properties ./
RUN gradle --no-daemon dependencies
COPY src ./src
RUN gradle --no-daemon clean build -x test

# Распаковываем fat-jar, чтобы jdeps увидел классы и библиотеки
RUN jar xf build/libs/DevOps-1.0.0.jar

# Анализируем зависимости
RUN jdeps \
    --ignore-missing-deps \
    -q \
    --recursive \
    --multi-release 21 \
    --print-module-deps \
    --class-path 'BOOT-INF/lib/*' \
    build/libs/DevOps-1.0.0.jar > deps.info

# Собираем минимальный JRE
RUN jlink \
    --add-modules $(cat deps.info) \
    --strip-debug \
    --compress zip-6 \
    --no-header-files \
    --no-man-pages \
    --output /custom-jre

# ── Финальный образ ────────────────────────────
FROM debian:bookworm-slim
ENV JAVA_HOME=/opt/java
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=build /custom-jre $JAVA_HOME
COPY --from=build /app/build/libs/DevOps-1.0.0.jar /app/app.jar
WORKDIR /app
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Разбор флагов jdeps

| Флаг | Зачем |
|------|-------|
| `--ignore-missing-deps` | Некоторые библиотеки ссылаются на необязательные зависимости. Без флага jdeps упадёт с ошибкой. |
| `-q` | Тихий режим — только результат, без отладочного вывода. |
| `--recursive` | Анализировать зависимости рекурсивно (dependency of dependency). |
| `--multi-release 21` | Корректно обрабатывать multi-release JAR'ы (библиотеки с разным кодом для разных версий Java). |
| `--print-module-deps` | Вывести список модулей через запятую — ровно то, что ожидает `jlink --add-modules`. |
| `--class-path 'BOOT-INF/lib/*'` | Путь к библиотекам внутри распакованного Spring Boot JAR. |

#### Разбор флагов jlink

| Флаг | Зачем |
|------|-------|
| `--add-modules $(cat deps.info)` | Включить только нужные модули (из вывода jdeps). |
| `--strip-debug` | Удалить отладочную информацию — экономит ~30% размера. |
| `--compress zip-6` | Сжать модули. В Java 21 формат изменился: `zip-0` (без сжатия) до `zip-9` (макс). В Java 17 и раньше — `--compress 2`. |
| `--no-header-files` | Убрать C-заголовки (нужны только для JNI-разработки). |
| `--no-man-pages` | Убрать man-страницы. |

> **Важно!** `--compress 2` работал в Java 17, но в Java 21 синтаксис поменялся на `--compress zip-6`. Если видите ошибку `Error: Invalid compression level` — проверьте версию JDK.

#### Сравнение размеров

| Подход | Размер образа |
|--------|--------------|
| Один этап, gradle + JDK | ~770МБ |
| Multi-stage, eclipse-temurin:21-jre | ~300МБ |
| Multi-stage, jlink + debian:slim | ~100-160МБ |
| Multi-stage, jlink + alpine | ~60-80МБ |

#### Когда НЕ стоит использовать jlink

jlink — это оптимизация, а не обязательный шаг. Если у вас образ на 300МБ и он вас устраивает — не трогайте. jlink усложняет Dockerfile и добавляет хрупкость: если библиотека через рефлексию загрузит класс из модуля, который jdeps не обнаружил, — приложение упадёт в runtime. Используйте jlink, когда размер образа критичен (edge-устройства, массовый деплой, жёсткие лимиты в Kubernetes).

---

## Урок 10. Слои Docker | BuildKit | Оптимизация сборки

> Этот урок вставляется **после** текущего Урока 9 (Multi-stage Build). Текущий Урок 10 (Лучшие практики) становится Уроком 11.

### Как устроены слои

Мы уже упоминали слои в предыдущих уроках. Сейчас разберём механизм подробно — это ключ к быстрым сборкам.

Каждая инструкция в Dockerfile (`FROM`, `RUN`, `COPY`, `ADD`) создаёт новый слой. Слой — это набор изменений в файловой системе: какие файлы добавлены, изменены, удалены.

```
┌────────────────────────────────┐
│  Layer 5: RUN mvn package      │  ← самый верхний слой
├────────────────────────────────┤
│  Layer 4: COPY src ./src       │
├────────────────────────────────┤
│  Layer 3: RUN mvn dep:offline  │
├────────────────────────────────┤
│  Layer 2: COPY pom.xml .       │
├────────────────────────────────┤
│  Layer 1: FROM maven:3.9-...   │  ← базовый образ (тоже набор слоёв)
└────────────────────────────────┘
```

Docker использует **Union Filesystem** (OverlayFS в современных версиях) — слои «накладываются» друг на друга. При чтении файла Docker ищет его сверху вниз. При записи — изменения идут только в самый верхний (writable) слой контейнера.

### Кэширование слоёв

Это самая важная оптимизация в Docker. Правило простое:

> Если инструкция и **все предыдущие инструкции** не изменились — Docker берёт результат из кэша.

Как только Docker встречает первое изменение — все последующие слои пересобираются **с нуля**.

```
COPY pom.xml .           ← pom.xml не изменился → кэш ✅
RUN mvn dep:offline      ← предыдущий слой из кэша → этот тоже ✅
COPY src ./src           ← изменили один .java файл → кэш сломан ❌
RUN mvn package          ← предыдущий слой пересобран → этот тоже ❌
```

Отсюда золотое правило:

> **Располагайте инструкции от редко меняющихся к часто меняющимся.**

Вот почему мы сначала копируем `pom.xml` и скачиваем зависимости, и только потом — исходники. Зависимости меняются раз в неделю, а код — каждый коммит. Без этого трюка Docker будет скачивать все зависимости при каждом изменении одной строки в коде.

### BuildKit

BuildKit — современный движок сборки Docker-образов. Он пришёл на смену «классическому» движку и включён по умолчанию начиная с Docker Engine 23.0 (2023).

Если у вас Docker Desktop или свежий Docker Engine — BuildKit уже работает. Проверить:

```bash
docker buildx version
```

Если по какой-то причине используется старый движок, включите BuildKit:

```bash
export DOCKER_BUILDKIT=1
```

#### Что даёт BuildKit

**1. Параллельная сборка**

В multi-stage build, если этапы не зависят друг от друга, BuildKit собирает их параллельно:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS backend
# ... сборка бэкенда ...

FROM node:22 AS frontend
# ... сборка фронтенда ...

FROM nginx:alpine
COPY --from=backend /app/target/*.jar /app/
COPY --from=frontend /app/dist /usr/share/nginx/html
```

Здесь этапы `backend` и `frontend` независимы — BuildKit запустит их одновременно.

**2. Умное кэширование**

BuildKit пропускает этапы, результат которых не нужен финальному образу. Если вы собираете только этап `backend` через `--target backend`, этап `frontend` не будет выполнен.

**3. Подробный вывод сборки**

По умолчанию BuildKit показывает компактный прогресс. Чтобы увидеть время каждого шага (полезно для оптимизации):

```bash
docker build --progress=plain -t myapp .
```

Пример вывода:

```
#5 [build 2/6] COPY pom.xml .
#5 DONE 0.1s

#6 [build 3/6] RUN mvn dependency:go-offline
#6 DONE 45.2s        ← вот где тратится время

#7 [build 4/6] COPY src ./src
#7 DONE 0.3s

#8 [build 5/6] RUN mvn package -DskipTests
#8 DONE 12.1s
```

Сразу видно, что скачивание зависимостей (45с) — самый долгий шаг. Именно его нужно кэшировать.

**4. Сборка без кэша**

Иногда кэш мешает (например, зависимость обновилась, но pom.xml не менялся):

```bash
docker build --no-cache -t myapp .
```

**5. Сборка конкретного этапа**

```bash
docker build --target build -t myapp-build .
```

Полезно для отладки: можно зайти в промежуточный образ и посмотреть, что получилось после сборки.

### docker history — анатомия образа

Чтобы понять, из чего состоит образ и сколько весит каждый слой:

```bash
docker history myapp
```

Пример:

```
IMAGE          CREATED          CREATED BY                                      SIZE
a1b2c3d4       2 minutes ago    ENTRYPOINT ["java" "-jar" "app.jar"]            0B
e5f6g7h8       2 minutes ago    COPY --from=build /app/target/*.jar app.jar     45MB
i9j0k1l2       3 minutes ago    WORKDIR /app                                    0B
m3n4o5p6       4 weeks ago      /bin/sh -c set -eux; ...                        175MB
```

Видим: базовый образ JRE весит 175МБ, наш JAR — 45МБ. Если образ подозрительно большой — `docker history` покажет, какой слой виноват.

### Практическая оптимизация

Типичная ошибка новичка — менять Dockerfile и удивляться, что сборка занимает 3 минуты каждый раз. Алгоритм диагностики:

1. Соберите с `--progress=plain` и найдите самые долгие шаги.
2. Проверьте `docker history` — нет ли подозрительно тяжёлых слоёв.
3. Убедитесь, что `.dockerignore` существует и содержит `.git`, `target`, `build`, `node_modules`.
4. Убедитесь, что зависимости кэшируются отдельно от исходников.
5. Проверьте, не копируете ли лишнее через `COPY . .` (без `.dockerignore`).

> **Из практики**: однажды сборка занимала 5 минут вместо обычных 30 секунд. Оказалось, `.dockerignore` отсутствовал, и Docker каждый раз отправлял в контекст 2ГБ папку `.git`. Один файл — и сборка снова 30 секунд.

### Задание

1. Для проекта из предыдущих уроков:
   - Соберите образ с флагом `--progress=plain`.
   - Найдите самый долгий шаг сборки. Запишите его время.
   - Внесите небольшое изменение в исходный код (например, добавьте пробел в любой .java-файл).
   - Соберите снова (без `--no-cache`). Сравните время — убедитесь, что слой с зависимостями взят из кэша.
2. Выполните `docker history <ваш_образ>`. Определите, сколько весит каждый слой.
3. Приложите вывод обеих команд.

---

## Урок 12. Docker в CI/CD (Jenkins)

> Этот урок добавляется **после** Урока 11 (Лучшие практики) как финальный урок модуля Docker.

### Зачем Docker в CI/CD

До этого мы собирали Docker-образы вручную на своей машине. В реальном проекте так не делают — образы собираются автоматически при каждом пуше в Git. Типичный пайплайн:

```
Push в Git → CI-сервер обнаруживает изменения → Тесты → Сборка образа → Push в Registry
```

В этом уроке мы настроим сборку Docker-образа внутри Jenkins.

### Как Jenkins запускает Docker

Jenkins работает на сервере (или в контейнере). Чтобы Jenkins мог выполнять `docker build`, ему нужен доступ к Docker Engine. Есть два подхода:

**1. Docker CLI на агенте** — Jenkins-агент имеет установленный Docker CLI и доступ к Docker socket.

**2. Docker-in-Docker (DinD)** — внутри контейнера Jenkins запускается отдельный Docker daemon. Этот подход сложнее и имеет проблемы с безопасностью. Мы используем первый.

### Подготовка Jenkins-агента

Агент Jenkins — это машина (или контейнер), на которой выполняются сборки. Нам нужен агент с JDK 21 и доступом к Docker.

**Запуск агента:**

```bash
docker run -d --name=agent-jdk21 \
  -p 2221:22 \
  --group-add $(getent group docker | cut -d: -f3) \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e "JENKINS_AGENT_SSH_PUBKEY=<ваш_SSH_ключ>" \
  jenkins/ssh-agent:latest-jdk21
```

Разберём ключевые моменты:

`--group-add $(getent group docker | cut -d: -f3)` — добавляет контейнер в группу `docker` хост-машины. Команда `getent group docker` возвращает что-то вроде `docker:x:999:`, а `cut -d: -f3` извлекает GID (999). Это нужно, чтобы процесс внутри контейнера мог обращаться к Docker socket.

`-v /var/run/docker.sock:/var/run/docker.sock` — пробрасываем Docker socket с хоста внутрь контейнера. Это и есть «мост» между агентом и Docker Engine хоста.

> **Что такое Docker socket?** Это Unix-сокет (`/var/run/docker.sock`), через который Docker CLI общается с Docker Daemon. Пробросив его в контейнер, мы даём контейнеру возможность управлять Docker'ом на хосте. Это не Docker-in-Docker — здесь один общий Docker daemon.

**Установка Docker CLI внутри агента:**

```bash
docker exec -u root -it agent-jdk21 bash

# Внутри контейнера:
apt-get update && apt-get install -y docker.io

# Проверяем GID группы docker на хосте
getent group docker
# Если пусто — создаём группу с тем же GID, что на хосте:
groupadd -g 999 docker      # 999 — GID с хоста
usermod -aG docker jenkins

# Проверяем
id jenkins
# uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins),999(docker)

exit

# На хосте — перезапускаем агент, чтобы группы применились
docker restart agent-jdk21
```

Теперь агент может выполнять `docker build`.

### Dockerfile для CI/CD

Когда проект собирается в Jenkins, Jenkins уже выполняет часть работы: клонирует репозиторий, запускает тесты, собирает JAR. Поэтому Dockerfile для CI/CD отличается от «самодостаточного» — ему не нужен этап сборки с Maven/Gradle:

```dockerfile
# Dockerfile (для CI/CD — JAR уже собран Jenkins'ом)
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY build/libs/*.jar app.jar

# Опционально: jlink для минимального JRE
RUN jar xf app.jar && \
    jdeps --ignore-missing-deps -q \
      --recursive --multi-release 21 \
      --print-module-deps \
      --class-path 'BOOT-INF/lib/*' \
      app.jar > deps.info && \
    jlink \
      --add-modules $(cat deps.info) \
      --strip-debug --compress zip-6 \
      --no-header-files --no-man-pages \
      --output /custom-jre

FROM debian:bookworm-slim
ENV JAVA_HOME=/opt/java
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=builder /custom-jre $JAVA_HOME
COPY --from=builder /app/app.jar /app/app.jar
WORKDIR /app
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Зачем два Dockerfile?** Один (полный, с этапом сборки) — для локальной разработки и `docker compose up`. Второй (без сборки) — для CI, где JAR уже собран. На практике часто обходятся одним полным Dockerfile и для CI — Jenkins просто вызывает `docker build` и всё.

### Jenkinsfile

```groovy
pipeline {
    agent { label 'agent-jdk21' }

    stages {
        stage('Prepare') {
            steps {
                sh 'chmod +x ./gradlew'
            }
        }
        stage('Test') {
            steps {
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
                sh 'docker build -t job4j_devops .'
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
                // Уведомление (Telegram, Slack и т.д.)
                echo info
            }
        }
    }
}
```

Этапы:
1. **Prepare** — делаем `gradlew` исполняемым (на Windows не нужно, но на Linux обязательно).
2. **Test** — запускаем тесты и проверки.
3. **Build JAR** — собираем артефакт.
4. **Docker Build** — собираем Docker-образ.

### Пуш образа в Registry

Собрать образ — половина дела. Его нужно залить в registry, откуда production-серверы смогут его скачать.

**Docker Hub:**

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
                docker tag job4j_devops $DOCKER_USER/job4j_devops:${BUILD_NUMBER}
                docker tag job4j_devops $DOCKER_USER/job4j_devops:latest
                docker push $DOCKER_USER/job4j_devops:${BUILD_NUMBER}
                docker push $DOCKER_USER/job4j_devops:latest
            '''
        }
    }
}
```

Обратите внимание:
- Учётные данные хранятся в Jenkins Credentials, а не в Jenkinsfile.
- Образ получает два тега: номер сборки (для отката) и `latest` (для удобства).
- `--password-stdin` — пароль передаётся через pipe, а не аргументом командной строки (безопасность).

> **Из практики**: всегда тегируйте образы номером сборки или хешем коммита. Тег `latest` — удобный указатель, но если что-то сломалось, вы должны мочь откатиться на конкретную версию: `docker pull myuser/myapp:42`.

### Полный пайплайн: от пуша до production

```
Developer pushes to Git
        │
        ▼
Jenkins detects change (webhook / poll)
        │
        ▼
┌─ Pipeline ─────────────────────┐
│  1. Checkout                   │
│  2. Test (gradlew check)       │
│  3. Build JAR (gradlew build)  │
│  4. Docker Build               │
│  5. Docker Push to Registry    │
│  6. Deploy (kubectl / ssh)     │
└────────────────────────────────┘
        │
        ▼
Production pulls new image
```

Шаг 6 (Deploy) выходит за рамки этого курса — он относится к Kubernetes или настройке production-сервера. Но важно понимать, что Docker-образ — это единица доставки. Jenkins его собирает, registry хранит, production скачивает и запускает.

### Безопасность Docker в CI/CD

Несколько правил, которые спасут от неприятностей:

**1. Не храните секреты в образе.** Никогда не делайте `COPY .env .` или `ENV DB_PASSWORD=secret` в Dockerfile. Любой, кто скачает образ, увидит секреты через `docker history` или `docker inspect`.

**2. Docker socket = root-доступ к хосту.** Пробросив `/var/run/docker.sock`, вы фактически дали контейнеру права root на хост-машине. Контейнер может запустить `docker run -v /:/host ubuntu rm -rf /host`. Пробрасывайте socket только доверенным агентам.

**3. Сканируйте образы.** Docker Scout (встроен в Docker Desktop) или Trivy (open-source) находят уязвимости в зависимостях:

```bash
# Docker Scout
docker scout quickview myapp:latest

# Trivy
trivy image myapp:latest
```

### Задание

1. Если у вас настроен Jenkins:
   - Добавьте в Jenkinsfile этап `Docker Build`.
   - Убедитесь, что Jenkins-агент имеет доступ к Docker.
   - Запустите сборку и приложите скриншот этапа Docker Build.
2. Если Jenkins не настроен:
   - Опишите текстом, какие этапы должен содержать CI/CD пайплайн для вашего Docker-проекта.
   - Объясните, зачем пробрасывать `/var/run/docker.sock` и какие риски это несёт.
3. Приложите ответ на проверку.
