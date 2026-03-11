# 3.6.5. Docker — обновлённые уроки (2026)

> **Целевой стек**: Docker Engine 27+ / Docker Desktop 4.x, Java 21 (Eclipse Temurin), Spring Boot 3.4, Maven 3.9+
>
> **Обозначения**: 🪟 Windows · 🍎 macOS · 🐧 Linux

---

## Урок 1. Виртуализация

### Что такое виртуализация

Виртуализация — технология, позволяющая запускать несколько изолированных вычислительных сред на одном физическом устройстве. Каждая среда ведёт себя как отдельная машина: со своей ОС, процессами, сетью и файловой системой.

Простой пример — вы на одном ноутбуке запускаете и рабочий Linux-сервер, и тестовую Windows-среду, и продакшн-копию базы данных. Все три не влияют друг на друга.

### Зачем это нужно

- **Изоляция** — сбой в одной среде не затрагивает другие.
- **Тиражируемость** — можно создать образ среды и передать коллеге. Он получит идентичное окружение.
- **Эффективность** — один сервер обслуживает десятки сервисов вместо одного.

### Два подхода к виртуализации

**Аппаратная виртуализация (гипервизор)**

Гипервизор (VMware, Hyper-V, VirtualBox) создаёт полноценные виртуальные машины (ВМ). Каждая ВМ содержит собственное ядро ОС, виртуальные устройства и полный набор системных библиотек.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App A   │  │  App B   │  │  App C   │
│  Libs    │  │  Libs    │  │  Libs    │
│  Guest OS│  │  Guest OS│  │  Guest OS│
└────┬─────┘  └────┬─────┘  └────┬─────┘
     └──────┬──────┴──────┬──────┘
        ┌───┴─────────────┴───┐
        │     Гипервизор      │
        ├─────────────────────┤
        │     Host OS         │
        ├─────────────────────┤
        │     Hardware        │
        └─────────────────────┘
```

Плюсы: полная изоляция, можно запускать разные ОС.
Минусы: тяжёлый — каждая ВМ занимает гигабайты, запускается минутами.

**Контейнерная виртуализация**

Контейнеры разделяют ядро хост-системы. Внутри контейнера — только приложение и его зависимости, без собственного ядра ОС.

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App A   │  │  App B   │  │  App C   │
│  Libs    │  │  Libs    │  │  Libs    │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     └──────┬──────┴──────┬──────┘
        ┌───┴─────────────┴───┐
        │   Container Runtime │
        │   (Docker Engine)   │
        ├─────────────────────┤
        │     Host OS (ядро)  │
        ├─────────────────────┤
        │     Hardware        │
        └─────────────────────┘
```

Плюсы: легковесный (мегабайты вместо гигабайт), запускается за секунды, идеален для микросервисов.
Минусы: контейнеры делят ядро хоста — нельзя запустить Windows-контейнер на Linux-хосте (и наоборот, без специальных средств).

### Контейнеры и Docker

Docker — самая популярная платформа контейнеризации. Основные понятия:

- **Образ (Image)** — неизменяемый снимок окружения: ОС + зависимости + приложение. Это «рецепт».
- **Контейнер (Container)** — запущенный экземпляр образа. Это «блюдо по рецепту». Контейнеров по одному образу можно создать сколько угодно.
- **Registry** — хранилище образов. Docker Hub — публичный реестр по умолчанию.

### OCI — открытый стандарт

В 2015 году Docker и другие компании создали **Open Container Initiative (OCI)** — набор открытых стандартов для контейнеров. Благодаря OCI, образы, собранные Docker, можно запускать в Podman, containerd, CRI-O и других runtime'ах. Это важно знать, потому что:

- Docker — не единственный инструмент, но самый популярный для разработки.
- В production-средах (Kubernetes) чаще используется containerd напрямую.
- Образы совместимы между всеми OCI-совместимыми runtime'ами.

### Экосистема Docker

```
┌─────────────────────────────────────────────────┐
│                  Developer Machine               │
│                                                  │
│  ┌──────────┐    ┌───────────────────────────┐  │
│  │ Docker   │───▶│      Docker Daemon        │  │
│  │ CLI      │    │  (dockerd)                │  │
│  └──────────┘    │                           │  │
│                  │  ┌─────────┐ ┌─────────┐  │  │
│                  │  │Container│ │Container│  │  │
│                  │  │   A     │ │   B     │  │  │
│                  │  └─────────┘ └─────────┘  │  │
│                  └───────────────────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │ pull / push
                ┌──────┴──────┐
                │  Registry   │
                │ (Docker Hub)│
                └─────────────┘
```

- **Docker CLI** — команды, которые вводит разработчик (`docker run`, `docker build` и т.д.).
- **Docker Daemon (dockerd)** — фоновый процесс, который управляет образами, контейнерами, сетями и томами.
- **Docker Desktop** — приложение для Windows / macOS / Linux, которое устанавливает и CLI, и Daemon, и Docker Compose, и GUI.

### Задание

Ознакомьтесь с материалом урока. Убедитесь, что понимаете разницу между аппаратной и контейнерной виртуализацией.

---

## Урок 2. Установка Docker

В этом уроке мы установим Docker. Инструкции даны для трёх платформ: Windows, macOS и Linux.

### Docker Desktop vs Docker Engine

- **Docker Desktop** — приложение с GUI, которое устанавливает Docker Engine, CLI, Docker Compose и Kubernetes. Подходит для Windows, macOS и Linux (desktop).
- **Docker Engine** — только движок + CLI. Устанавливается напрямую на Linux-сервера.

> **Лицензия**: Docker Desktop бесплатен для индивидуального использования, образования и небольших компаний (< 250 сотрудников и < $10M выручки). Для крупных компаний — платная подписка. Docker Engine — полностью бесплатный (Apache 2.0).

### 🪟 Windows

**Требования:**
- Windows 10 22H2 (build 19045) / Windows 11 23H2 или выше
- Включённая виртуализация в BIOS (Intel VT-x / AMD-V)
- WSL 2

**Шаг 1: Установка WSL 2**

Откройте PowerShell от имени администратора:

```powershell
wsl --install
```

Перезагрузите компьютер. После перезагрузки WSL автоматически установит Ubuntu.

Проверьте:

```powershell
wsl --version
```

**Шаг 2: Установка Docker Desktop**

1. Скачайте установщик: https://www.docker.com/products/docker-desktop/
2. Запустите `Docker Desktop Installer.exe`
3. В настройках убедитесь, что выбрано **Use WSL 2 instead of Hyper-V**
4. Нажмите **OK**, дождитесь окончания установки
5. Перезагрузите компьютер

**Шаг 3: Проверка**

Откройте PowerShell или Windows Terminal:

```bash
docker --version
docker compose version
docker run hello-world
```

### 🍎 macOS

**Требования:**
- macOS 13 (Ventura) или новее
- Apple Silicon (M1/M2/M3/M4) или Intel

**Шаг 1: Установка Docker Desktop**

1. Скачайте установщик: https://www.docker.com/products/docker-desktop/
   - Выберите вариант для вашего чипа: **Apple chip** или **Intel chip**
2. Откройте `.dmg` файл, перетащите Docker в Applications
3. Запустите Docker из Applications

Альтернатива через Homebrew:

```bash
brew install --cask docker
```

**Шаг 2: Проверка**

Откройте Terminal:

```bash
docker --version
docker compose version
docker run hello-world
```

### 🐧 Linux (Ubuntu / Debian)

На Linux можно установить либо Docker Desktop (с GUI), либо Docker Engine напрямую. Для разработки рекомендуется Docker Engine — он легче и не требует GUI.

**Шаг 1: Удалите старые версии (если есть)**

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

**Шаг 2: Установка через официальный скрипт**

Самый простой способ:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Или вручную через apt (для Ubuntu 22.04+):

```bash
# Добавляем GPG-ключ Docker
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Добавляем репозиторий
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Устанавливаем
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

**Шаг 3: Настройка прав (чтобы не использовать sudo)**

```bash
sudo groupadd docker          # может уже существовать
sudo usermod -aG docker $USER
newgrp docker                  # применить без перезагрузки
```

**Шаг 4: Проверка**

```bash
docker --version
docker compose version
docker run hello-world
```

### Docker Compose — уже встроен

Начиная с Docker Engine 20.10+ и Docker Desktop 4.x, Docker Compose v2 идёт в комплекте как плагин. Вызывается командой `docker compose` (без дефиса).

Старая команда `docker-compose` (через дефис, v1) **deprecated с июля 2023** и больше не поддерживается.

```bash
# ✅ Правильно (v2)
docker compose up

# ❌ Устарело (v1)
docker-compose up
```

### Задание

1. Установите Docker на вашу ОС, следуя инструкции выше.
2. Выполните `docker --version` и `docker compose version`.
3. Выполните `docker run hello-world`.
4. Скопируйте вывод всех трёх команд и отправьте на проверку.

---

## Урок 3. Первые шаги с Docker

В этом уроке мы познакомимся с основными командами Docker CLI и убедимся, что всё работает корректно.

### Проверяем окружение

```bash
docker info
```

Эта команда выведет информацию о Docker: версию, количество контейнеров, образов, storage driver и т.д.

### Запускаем первый контейнер

Так как мы Java-разработчики, запустим контейнер с JDK:

```bash
docker run -it --rm eclipse-temurin:21-jdk bash
```

> **Важно!** Образ `openjdk` считается устаревшим (deprecated). Актуальный образ для Java — `eclipse-temurin` от Eclipse Adoptium.

Разберём команду:
- `docker run` — создать контейнер из образа и запустить его
- `-i` — интерактивный режим (stdin остаётся открытым)
- `-t` — выделить псевдотерминал (TTY)
- `--rm` — автоматически удалить контейнер после выхода
- `eclipse-temurin:21-jdk` — образ и тег (Java 21, JDK)
- `bash` — команда для запуска внутри контейнера

### Проверяем окружение внутри контейнера

```bash
# Версия ОС
cat /etc/os-release

# Версия Java
java --version

# Запускаем jshell
jshell
```

В jshell:

```java
System.out.println("Hello from Docker!")
/exit
```

Для выхода из контейнера:

```bash
exit
```

Так как мы указали `--rm`, контейнер автоматически удалится.

### Полезные команды для знакомства

```bash
# Список скачанных образов
docker images

# Список запущенных контейнеров
docker ps

# Список всех контейнеров (включая остановленные)
docker ps -a

# Информация об использовании диска Docker'ом
docker system df

# Получить справку по любой команде
docker run --help
```

### Как Docker находит образы

Когда вы выполняете `docker run eclipse-temurin:21-jdk`, Docker:

1. Проверяет, есть ли образ локально.
2. Если нет — скачивает с Docker Hub (https://hub.docker.com).
3. Создаёт контейнер на основе образа.
4. Запускает в нём указанную команду.

Если тег не указан, Docker использует тег `latest`.

### Именование образов

Полный формат: `[registry/][namespace/]name[:tag]`

```
eclipse-temurin:21-jdk          ← Docker Hub, библиотека, тег 21-jdk
postgres:17                     ← Docker Hub, библиотека, тег 17
mycompany/myapp:1.0.0           ← Docker Hub, namespace mycompany
ghcr.io/owner/app:latest        ← GitHub Container Registry
```

### Задание

1. Запустите контейнер `eclipse-temurin:21-jdk`, выполните `java --version` и сделайте скриншот.
2. Запустите контейнер `ubuntu`, выполните `cat /etc/os-release` и сделайте скриншот.
3. Выполните `docker images` и `docker ps -a`. Объясните, почему один контейнер остался в списке, а другой нет.
4. Отправьте скриншоты и объяснение на проверку.

---

## Урок 4. Управление жизненным циклом контейнера

В этом уроке мы научимся создавать, запускать, останавливать и удалять контейнеры.

> **Важно!** Docker-контейнер живёт, пока в нём есть хотя бы один работающий процесс. Когда процесс завершается — контейнер останавливается.

### docker run

Самая частая команда. Она объединяет pull + create + start (+ attach, если указаны -i/-t).

```bash
docker run -it eclipse-temurin:21-jdk bash
```

Внутри контейнера попробуем:

```bash
java --version
jshell
```

В jshell:

```java
System.out.println("Hello from job4j to Docker!")
/exit
```

Выходим из контейнера:

```bash
exit
```

### docker ps / docker container ls

```bash
# Активные контейнеры
docker ps

# Все (включая остановленные)
docker ps -a
```

После `exit` контейнер остановлен, но не удалён — он видён в `docker ps -a`.

### Жизненный цикл контейнера

```
         pull         create        start        (работает)
Registry ──────▶ Image ──────▶ Created ──────▶ Running
                                                  │
                                    ┌─────────────┤
                                    │             │
                                  pause        stop/kill
                                    │             │
                                    ▼             ▼
                                  Paused      Stopped (Exited)
                                    │             │
                                  unpause      start
                                    │             │
                                    ▼             │
                                  Running ◀───────┘
                                    │
                                   rm
                                    │
                                    ▼
                                 Removed
```

1. **pull** — скачивание образа из registry (если его нет локально)
2. **create** — создание контейнера на основе образа
3. **start** — запуск контейнера
4. **attach** — подключение к запущенному контейнеру (или работа в фоне — detach)
5. **stop / pause** — остановка. `stop` завершает процесс (SIGTERM, затем SIGKILL). `pause` «замораживает» процесс.
6. **rm** — удаление остановленного контейнера

### docker container start / attach

Запустим ранее остановленный контейнер:

```bash
# Узнаём id или name
docker ps -a

# Запускаем
docker container start <id_или_name>

# Проверяем, что контейнер активен
docker ps

# Подключаемся
docker container attach <id_или_name>
```

### Выход без остановки контейнера

Внутри контейнера нажмите **Ctrl+P, затем Ctrl+Q** (удерживая Ctrl, последовательно P и Q). Контейнер продолжит работу в фоне.

### docker run --rm

Флаг `--rm` говорит Docker: «удали контейнер сразу после остановки».

```bash
docker run --rm -it eclipse-temurin:21-jdk bash
# ... работаем ...
exit
docker ps -a   # контейнера нет в списке
```

Для учебных целей и одноразовых задач всегда используйте `--rm`.

### docker container rm

Удаление конкретного контейнера:

```bash
docker container rm <id_или_name>
```

Удаление всех остановленных контейнеров:

```bash
docker container prune
```

> В старых туториалах можно встретить `docker container rm $(docker container ls -aq)` — это то же самое, но `prune` удобнее.

### Полный цикл через отдельные команды

Разберём жизненный цикл поэтапно:

```bash
# 1. Создаём контейнер (без запуска)
docker container create --name job4j_docker -it eclipse-temurin:21-jdk bash

# 2. Запускаем
docker container start job4j_docker

# 3. Подключаемся
docker container attach job4j_docker

# 4. Внутри: java --version
#    Выход без остановки: Ctrl+P, Ctrl+Q

# 5. Проверяем — контейнер активен
docker ps

# 6. Останавливаем
docker container stop job4j_docker

# 7. Удаляем
docker container rm job4j_docker
```

### Теги образов

Один образ может иметь несколько тегов (версий):

```bash
# Java 21 на базе Ubuntu
docker run -it --rm eclipse-temurin:21-jdk bash

# Java 21 на базе Alpine (минимальный образ ~200МБ вместо ~400МБ)
docker run -it --rm eclipse-temurin:21-jdk-alpine sh

# Java 21, только JRE (без компилятора)
docker run -it --rm eclipse-temurin:21-jre bash
```

Посмотреть все доступные теги: https://hub.docker.com/_/eclipse-temurin/tags

> **Совет:** В production всегда указывайте конкретный тег (`eclipse-temurin:21.0.6_7-jre`), а не generic (`eclipse-temurin:21`). Тег `latest` может неожиданно обновиться.

### Задание

1. Выполните команды из урока.
2. Выберите любой образ из Docker Hub (не `eclipse-temurin`).
3. Для этого образа:
   - 3.1. Запустите контейнер через `docker run`. Внутри выполните любую команду.
   - 3.2. Повторите через последовательность `create → start → attach → stop → rm`.
4. Приложите скриншоты к заданию.

---

## Урок 5. Управление образами

### Просмотр образов

```bash
docker images
```

Вывод покажет: репозиторий, тег, ID, дату создания и размер.

### Скачивание образа

```bash
docker pull ubuntu
docker pull eclipse-temurin:21-jdk-alpine
```

Образы скачиваются послойно (layers). Если два образа используют общий слой — он скачивается только один раз.

### Удаление образа

```bash
docker rmi ubuntu
```

Если существуют контейнеры, использующие этот образ:

```bash
docker rmi --force ubuntu
```

### Очистка неиспользуемых образов

```bash
# Удалить «висячие» образы (без тега, промежуточные слои)
docker image prune

# Удалить все образы, которые не используются ни одним контейнером
docker image prune -a
```

### Сборка образа из контейнера (docker commit)

Это учебный способ. В реальных проектах используют Dockerfile (следующий урок).

Соберём образ программы, складывающей два числа.

**1. Создаём файл `Program.java` на хосте:**

```java
public class Program {
    public static void main(String[] args) {
        int a = Integer.parseInt(args[0]);
        int b = Integer.parseInt(args[1]);
        System.out.println("Sum: " + (a + b));
    }
}
```

**2. Запускаем контейнер в фоновом режиме:**

```bash
docker run -d -it --name builder eclipse-temurin:21-jdk bash
```

Здесь комбинация флагов `-d -it` выглядит противоречиво, но работает строго логично:

- `-d` (detach) — запустить контейнер в фоне, не подключаясь к нему.
- `-i` (interactive) — держать stdin открытым.
- `-t` (tty) — выделить псевдотерминал.

Зачем `-it`, если мы всё равно отсоединяемся? Без них процесс `bash` обнаружит, что ему не выделен терминал и нет потока ввода, — и мгновенно завершится. А раз процесс завершился — контейнер тоже. Флаги `-it` заставляют bash «ждать ввода» — процесс жив, контейнер жив. Позже мы подключимся к нему через `docker attach`.

**3. Копируем файл в контейнер:**

```bash
docker cp Program.java builder:/
```

**4. Подключаемся, компилируем и проверяем:**

```bash
docker container attach builder

javac Program.java
java Program 3 7
# Sum: 10
```

Выход без остановки: **Ctrl+P, Ctrl+Q**

**5. Создаём образ из контейнера:**

```bash
docker commit --change='ENTRYPOINT ["java", "Program"]' builder local:job4j
```

- `--change` позволяет вписать инструкцию, которая будет выполняться при запуске контейнера.
- `ENTRYPOINT ["java", "Program"]` — эквивалентно команде `java Program` в терминале.
- `local` — имя образа, `job4j` — тег.

**6. Запускаем свой образ:**

```bash
docker run --rm local:job4j 1 2
# Sum: 3
```

Аргументы `1` и `2` передаются в `args[]` нашей программы.

**7. Убираем за собой:**

```bash
docker container stop builder
docker container rm builder
```

### Задание

1. На основе образа `eclipse-temurin:21-jdk` собрать образ с программой:
   - Программа считает количество вхождений слова в текстовом файле.
   - Файл с программой и файл со словами предварительно перенести в контейнер через `docker cp`.
   - Слово передаётся аргументом командной строки.
   - Программа запускается через `docker run`.
2. Приложите скриншоты с запуском образа.

---

## Урок 6. Dockerfile

В этом уроке мы научимся создавать образы декларативно — через Dockerfile.

### Зачем Dockerfile

В предыдущем уроке мы собирали образ вручную через `docker commit`. Это неудобно: шаги не задокументированы, воспроизвести сборку сложно. Dockerfile решает эту проблему — он описывает сборку в виде текстового файла, который можно хранить в Git.

### Пример проекта

Возьмём элементарный Maven-проект. Исходный код: https://github.com/peterarsentev/job4j_docker

В корне проекта создаём **Dockerfile**:

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY . .
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/main.jar app.jar
CMD ["java", "-jar", "app.jar"]
```

> **Внимание!** Это уже многоэтапная сборка (multi-stage build). Мы подробнее разберём её в Уроке 9. Пока достаточно знать основные инструкции.

### Инструкции Dockerfile

| Инструкция | Назначение |
|-----------|-----------|
| `FROM` | Базовый образ. Может быть несколько (multi-stage). |
| `WORKDIR` | Установить рабочую директорию. Все последующие команды выполняются из неё. |
| `COPY` | Скопировать файлы с хоста в образ. |
| `RUN` | Выполнить команду **при сборке** образа. |
| `CMD` | Команда по умолчанию **при запуске** контейнера. |
| `ENTRYPOINT` | Точка входа при запуске. В отличие от CMD, аргументы `docker run` дописываются, а не заменяют. |
| `ENV` | Установить переменную окружения. |
| `EXPOSE` | Документирует порт (не открывает его!). |
| `ARG` | Аргумент сборки, доступен только во время `docker build`. |

### RUN vs CMD vs ENTRYPOINT

- **RUN** — выполняется при **сборке** образа. Результат сохраняется в слой образа.
- **CMD** — выполняется при **запуске** контейнера. Если передать аргументы в `docker run`, CMD заменяется.
- **ENTRYPOINT** — тоже при запуске, но аргументы `docker run` **дописываются** к ENTRYPOINT.

```dockerfile
# CMD: docker run myimage         → java -jar app.jar
#      docker run myimage bash    → bash (CMD заменён)
CMD ["java", "-jar", "app.jar"]

# ENTRYPOINT: docker run myimage         → java -jar app.jar
#             docker run myimage --debug  → java -jar app.jar --debug
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Сборка и запуск

```bash
# Клонируем проект
git clone https://github.com/peterarsentev/job4j_docker.git
cd job4j_docker

# Собираем образ (-t задаёт имя и тег)
docker build -t job4j_docker .

# Точка в конце — контекст сборки (текущая директория)

# Проверяем
docker images

# Запускаем
docker run --rm job4j_docker
```

### Плагин для сборки JAR

Для Spring Boot проектов используется `spring-boot-maven-plugin` (входит в `spring-boot-starter-parent`):

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

Для обычных Maven-проектов (не Spring Boot) — `maven-shade-plugin`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.6.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <finalName>main</finalName>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>ru.job4j.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Задание

1. Возьмите проект `job4j_di` (из блока Spring).
   - Если используете Spring Boot: убедитесь, что `spring-boot-maven-plugin` подключён.
   - Если нет: добавьте `maven-shade-plugin`.
2. Создайте Dockerfile в корне проекта.
3. Соберите образ и убедитесь, что он запускается.
4. Оставьте ссылку на коммит.

---

## Урок 7. Docker Compose

Docker Compose позволяет описать и запустить систему из нескольких контейнеров одной командой.

### Когда нужен Compose

Микросервисная архитектура подразумевает:
- Каждый сервис — отдельный контейнер.
- У каждого сервиса своя БД — ещё контейнер.
- Сервисы общаются по сети — нужна настройка сети.
- Порядок запуска важен (БД раньше приложения).

Всё это описывается в одном файле **`compose.yaml`** (рекомендуемое имя) или `docker-compose.yml` (устаревшее, но работает).

### Инструменты оркестрации Docker

| Инструмент | Назначение |
|-----------|-----------|
| **Dockerfile** | Сборка одного образа |
| **Docker Compose** | Запуск нескольких контейнеров на **одной** машине |
| **Kubernetes** | Запуск контейнеров на **кластере** из нескольких машин |

> Docker Swarm (аналог Kubernetes от Docker) существует, но в production-средах практически вытеснен Kubernetes.

### Пример: приложение + PostgreSQL

Рассмотрим на примере проекта job4j_tracker (Spring Boot 3.4 + PostgreSQL).

**Dockerfile** (в корне проекта):

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**compose.yaml**:

```yaml
services:
  db:
    image: postgres:17
    container_name: tracker_db
    environment:
      POSTGRES_DB: tracker
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - tracker_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: job4j_tracker
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/tracker
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: password
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
```

### Разбор compose.yaml

**Нет поля `version`!** Начиная с Docker Compose v2, поле `version` устарело и игнорируется. Если его указать — Docker выдаст warning.

Ключевые элементы:

- **services** — раздел, где описываются контейнеры.
- **image** — готовый образ из registry.
- **build** — собрать образ из Dockerfile.
- **environment** — переменные окружения внутри контейнера.
- **ports** — проброс портов `хост:контейнер`.
- **volumes** — монтирование томов (для персистентности данных).
- **depends_on** — порядок запуска. С `condition: service_healthy` Docker дождётся, пока зависимость пройдёт healthcheck.
- **healthcheck** — проверка «живости» контейнера.
- **restart** — политика перезапуска: `no`, `always`, `on-failure`, `unless-stopped`.

### Docker-сети: как контейнеры общаются

Контейнеры изолированы не только по файловой системе и процессам, но и по сети. Чтобы контейнеры могли общаться друг с другом (или с внешним миром), Docker предоставляет механизм виртуальных сетей.

#### Типы сетей (network drivers)

```
┌─────────────────────────────────────────────────────────────┐
│  Host Machine                                               │
│                                                             │
│  ┌─── bridge (default) ──────────────────────────────────┐  │
│  │  172.17.0.0/16                                        │  │
│  │  ┌───────────┐  ┌───────────┐                         │  │
│  │  │ Container │  │ Container │  ← контейнеры видят     │  │
│  │  │ 172.17.0.2│  │ 172.17.0.3│    друг друга по IP     │  │
│  │  └───────────┘  └───────────┘                         │  │
│  └───────────────────────┬───────────────────────────────┘  │
│                          │ docker0 (виртуальный мост)        │
│  ┌─── my_network (user-defined bridge) ──────────────────┐  │
│  │  ┌───────────┐  ┌───────────┐                         │  │
│  │  │ app       │  │ db        │  ← контейнеры видят     │  │
│  │  │           │──│           │    друг друга по ИМЕНИ   │  │
│  │  └───────────┘  └───────────┘                         │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Physical NIC (eth0 / en0 / Wi-Fi)                          │
└─────────────────────────────────────────────────────────────┘
```

| Драйвер | Описание | Когда использовать |
|---------|---------|-------------------|
| **bridge** | Виртуальный мост (default). Контейнеры подключаются к изолированной подсети. | Стандартная разработка, Docker Compose |
| **host** | Контейнер использует сеть хоста напрямую (без изоляции). | Максимальная производительность сети (только 🐧 Linux) |
| **none** | Без сети вообще. | Полная изоляция, batch-задачи |
| **overlay** | Сеть между несколькими Docker-хостами. | Docker Swarm, Kubernetes |

#### Default bridge vs User-defined bridge

Когда вы запускаете `docker run` без `--network`, контейнер попадает в **default bridge** (`docker0`). В этой сети контейнеры видят друг друга **только по IP-адресу**, DNS-имена не работают.

**User-defined bridge** (пользовательская сеть) — это то, что создаёт Docker Compose. В ней:
- контейнеры находят друг друга **по имени** (встроенный DNS);
- сеть изолирована от контейнеров в других проектах;
- можно подключать/отключать контейнеры на лету.

```bash
# Ручное создание сети
docker network create my_network

# Запуск контейнеров в этой сети
docker run -d --name db --network my_network postgres:17
docker run -d --name app --network my_network eclipse-temurin:21-jdk

# Теперь app может обратиться к db по имени:
docker exec app ping db   # работает!

# Список сетей
docker network ls

# Детали сети (какие контейнеры подключены, IP-адреса)
docker network inspect my_network

# Удаление
docker network rm my_network
```

#### Проброс портов (-p / ports)

Контейнер по умолчанию недоступен с хоста. Чтобы «пробросить» порт наружу:

```bash
# Формат: -p ХОСТ:КОНТЕЙНЕР
docker run -d -p 8080:8080 myapp        # localhost:8080 → контейнер:8080
docker run -d -p 5433:5432 postgres:17   # localhost:5433 → контейнер:5432
docker run -d -p 127.0.0.1:8080:8080 myapp  # только localhost, не все интерфейсы
```

В `compose.yaml`:

```yaml
services:
  app:
    ports:
      - "8080:8080"          # хост:контейнер
  db:
    ports:
      - "5432:5432"
```

> **Важно**: `EXPOSE` в Dockerfile **не пробрасывает** порт. Он лишь документирует, какой порт использует приложение. Реальный проброс делается через `-p` или `ports`.

#### Сеть в Docker Compose

Docker Compose автоматически создаёт user-defined bridge для каждого проекта. Имя сети: `<имя_папки>_default`. Все сервисы в `compose.yaml` подключены к этой сети.

Контейнеры обращаются друг к другу **по имени сервиса**:

```
jdbc:postgresql://db:5432/tracker
                  ^^
                  имя сервиса из compose.yaml
```

Docker DNS разрешает имя `db` в IP-адрес контейнера. Не нужно знать никаких IP — только имена сервисов.

#### Кастомные сети в Compose

Иногда нужно разделить сервисы по сетям (например, фронтенд не должен видеть БД напрямую):

```yaml
services:
  nginx:
    image: nginx
    networks:
      - frontend

  app:
    build: .
    networks:
      - frontend
      - backend

  db:
    image: postgres:17
    networks:
      - backend

networks:
  frontend:
  backend:
```

В этом примере:
- `nginx` видит `app`, но **не видит** `db`.
- `app` видит и `nginx`, и `db` (он в обеих сетях).
- `db` видит только `app`.

#### Полезные команды для отладки сети

```bash
# Какие сети существуют
docker network ls

# Какие контейнеры в сети, их IP-адреса
docker network inspect <network_name>

# Проверить связь между контейнерами
docker exec -it app ping db
docker exec -it app curl http://api:8080/health

# DNS-резолв внутри контейнера
docker exec -it app nslookup db
```

### Команды Docker Compose

```bash
# Сборка образов
docker compose build

# Запуск всех сервисов (в фоне)
docker compose up -d

# Запуск конкретного сервиса
docker compose up -d db

# Логи
docker compose logs
docker compose logs -f app    # follow конкретного сервиса

# Статус
docker compose ps

# Остановка
docker compose stop

# Остановка + удаление контейнеров и сетей
docker compose down

# Остановка + удаление контейнеров, сетей И томов (осторожно — данные БД удалятся!)
docker compose down -v
```

### Именованные тома (volumes)

В конце `compose.yaml` нужно объявить именованные тома:

```yaml
services:
  db:
    # ...
    volumes:
      - tracker_data:/var/lib/postgresql/data

volumes:
  tracker_data:
```

Это обеспечивает персистентность данных PostgreSQL между перезапусками контейнера.

### Spring Boot и переменные окружения

Spring Boot автоматически подхватывает переменные окружения. Переменная `SPRING_DATASOURCE_URL` переопределяет свойство `spring.datasource.url` из `application.properties`. Правило маппинга:

```
spring.datasource.url  →  SPRING_DATASOURCE_URL
точки → подчёркивания, всё в верхний регистр
```

Поэтому загружать переменные вручную (как было в оригинальном уроке через `System.getenv()`) **не нужно** — Spring Boot делает это сам.

### Задание

1. В корне проекта CheckDev (или вашего Spring Boot проекта) создайте `compose.yaml`.
   - Для каждого сервиса создайте Dockerfile.
   - Передайте настройки БД через переменные окружения.
2. Соберите проект: `docker compose build`
3. Запустите: `docker compose up -d`
4. Убедитесь:
   - Сервис отвечает на запросы с хост-системы.
   - После `docker compose down` и повторного `docker compose up -d` данные в БД сохранены (том работает).
5. Проверьте сеть:
   - Выполните `docker network ls` и найдите сеть, созданную Compose. Приложите скриншот.
   - Выполните `docker network inspect <имя_сети>` и убедитесь, что оба контейнера (app и db) подключены. Приложите скриншот.
   - Зайдите внутрь контейнера app (`docker exec -it <container> bash`) и выполните `ping db` — убедитесь, что DNS-резолв работает.
6. Оставьте ссылку на коммит.
7. Добавьте в README инструкцию по запуску через Docker Compose.

---

## Урок 8. Docker томы (Volumes)

### Проблема

Контейнеры по умолчанию **эфемерны**: все изменения внутри контейнера исчезают при его удалении.

Рассмотрим:

```bash
# Создаём контейнер, записываем файл
docker run -it --name demo ubuntu bash
echo "hello" > /data.txt
exit

# Перезапускаем — файл на месте
docker start demo
docker exec demo cat /data.txt   # hello

# Удаляем контейнер — файл потерян навсегда
docker rm demo
```

Три проблемы:
1. При использовании `--rm` данные теряются сразу.
2. Новый контейнер не видит данных предыдущего.
3. Один контейнер не видит изменений другого.

### Docker Volume (том)

Том — управляемая Docker'ом директория на хосте, которая монтируется внутрь контейнера.

```bash
# Создаём том
docker volume create job4j

# Список томов
docker volume ls

# Информация о томе
docker volume inspect job4j
```

### Использование тома

```bash
# Запускаем контейнер с томом
docker run -it --rm -v job4j:/data --name job4j_ubuntu ubuntu bash

# Создаём файл
echo "persistent data" > /data/hello.txt
exit

# Запускаем НОВЫЙ контейнер с тем же томом
docker run -it --rm -v job4j:/data --name job4j_ubuntu ubuntu bash
cat /data/hello.txt   # persistent data!
exit
```

Ключевой момент: `-v job4j:/data` монтирует том `job4j` в директорию `/data` внутри контейнера.

> **Совет**: если том не создан заранее, Docker создаст его автоматически.

### Совместный доступ к тому

Два контейнера могут одновременно использовать один том:

```bash
# Терминал 1
docker run -it --rm -v job4j:/shared --name writer ubuntu bash
echo "I like you" > /shared/message.txt

# Терминал 2
docker run -it --rm -v job4j:/shared --name reader ubuntu bash
cat /shared/message.txt   # I like you
```

### Типы монтирования

| Тип | Синтаксис | Когда использовать |
|-----|----------|-------------------|
| **Named Volume** | `-v myvolume:/path` | Данные БД, постоянное хранение |
| **Bind Mount** | `-v /host/path:/container/path` | Разработка — монтируем исходники |
| **tmpfs** | `--tmpfs /path` | Временные данные в памяти |

### PostgreSQL с томом

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=tracker \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17
```

Разберём:
- `-d` — фоновый режим
- `-e` — переменные окружения
- `-p 5432:5432` — проброс порта
- `-v pgdata:/var/lib/postgresql/data` — данные PostgreSQL хранятся в томе

Проверяем:

```bash
# Подключаемся к PostgreSQL внутри контейнера
docker exec -it postgres psql -U postgres

# Создаём таблицу
CREATE DATABASE tracker;
\c tracker
CREATE TABLE items(id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO items(name) VALUES ('test');
\q

# Удаляем контейнер
docker stop postgres
docker rm postgres

# Создаём новый контейнер с тем же томом
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17

# Данные на месте!
docker exec -it postgres psql -U postgres -d tracker -c "SELECT * FROM items;"
```

### Подключение к контейнеру PostgreSQL с хоста

Так как мы пробросили порт (`-p 5432:5432`), к базе можно подключиться с хост-машины любым клиентом: pgAdmin, DBeaver, IntelliJ IDEA Database Tools или `psql` (если установлен):

```bash
psql -h localhost -p 5432 -U postgres -d tracker
```

### Очистка томов

```bash
# Удалить конкретный том
docker volume rm job4j

# Удалить все неиспользуемые тома (осторожно!)
docker volume prune
```

### Задание

1. Создайте том. Запустите контейнер Ubuntu с этим томом, создайте файл. Удалите контейнер, создайте новый с тем же томом. Убедитесь, что файл на месте. Приложите скриншот.
2. Запустите PostgreSQL с томом. Создайте базу и таблицу. Удалите контейнер, запустите новый с тем же томом. Убедитесь, что данные сохранены. Приложите скриншот.
3. Подключитесь к БД контейнера с хоста (через DBeaver, pgAdmin или psql). Приложите скриншот.

---

## Урок 9. Многоэтапная сборка (Multi-stage Build)

### Проблема

Для сборки Java-проекта нужны Maven + JDK. Для запуска — только JRE. Если мы используем один этап сборки, в итоговый образ попадёт и Maven, и все зависимости из `.m2`, и исходники. Образ получится тяжёлым (~800МБ+).

### Решение: Multi-stage Build

Идея: разделить сборку на этапы. Каждый этап начинается с `FROM`. Из предыдущих этапов можно копировать только нужные файлы.

```dockerfile
# ── Этап 1: сборка ─────────────────────────────
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# ── Этап 2: запуск ─────────────────────────────
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Этап 1** (`build`):
- Берём тяжёлый образ с Maven + JDK.
- Сначала копируем только `pom.xml` и скачиваем зависимости. Это **кэширование слоёв** — если pom.xml не менялся, Docker возьмёт зависимости из кэша.
- Затем копируем исходники и собираем JAR.

**Этап 2** (runtime):
- Берём лёгкий образ с JRE (без Maven, без компилятора).
- Копируем только готовый JAR из этапа `build`.
- Указываем, как запускать.

**Результат**: итоговый образ содержит только JRE + JAR. Размер ~300МБ вместо ~800МБ.

### Выбор базового образа

| Образ | Размер | Содержит | Когда использовать |
|-------|--------|----------|-------------------|
| `eclipse-temurin:21-jdk` | ~400МБ | JDK + JRE | Этап сборки (если без Maven) |
| `eclipse-temurin:21-jre` | ~270МБ | Только JRE | Этап запуска (рекомендуется) |
| `eclipse-temurin:21-jre-alpine` | ~100МБ | JRE + Alpine Linux | Минимальный размер |

> **Примечание**: Alpine-образы значительно легче, но используют musl вместо glibc. Для большинства Spring Boot приложений это не проблема, но тестируйте!

### jlink — ещё легче

Для продвинутых: Java 9+ позволяет через `jlink` собрать кастомный JRE только с нужными модулями:

```dockerfile
FROM eclipse-temurin:21-jdk AS jre-builder
RUN jlink \
  --add-modules java.base,java.logging,java.sql,java.naming,java.net.http,java.management \
  --strip-debug --no-man-pages --no-header-files \
  --output /custom-jre

FROM debian:bookworm-slim
COPY --from=jre-builder /custom-jre /opt/java
COPY --from=build /app/target/*.jar /app/app.jar
ENV PATH="/opt/java/bin:$PATH"
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Итоговый образ может весить ~100МБ. Но модули нужно подбирать вручную — если не хватит модуля, приложение упадёт в runtime.

### Обновление compose.yaml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: shortcut
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
```

Поле `image` заменяется на `build` — Docker Compose сам соберёт образ из Dockerfile.

### Сборка и запуск

```bash
# Собрать образы
docker compose build

# Запустить в фоне
docker compose up -d

# Проверить логи
docker compose logs -f app
```

Весь проект собирается и запускается двумя командами, без предварительной установки Java или Maven на хосте.

### Задание

1. Для проектов, в которые вы добавили поддержку Docker:
   - Обновите Dockerfile: используйте multi-stage build.
   - Обновите `compose.yaml`.
   - Убедитесь, что проект собирается и запускается командами `docker compose build` и `docker compose up -d`.
   - Обновите README.md.
2. Залейте изменения и приложите ссылку.

---

## Урок 10. Docker: лучшие практики

В этом уроке собраны практические рекомендации по работе с Docker. Их соблюдение поможет создавать надёжные, безопасные и эффективные образы.

### 1. Используйте .dockerignore

Аналог `.gitignore`, но для Docker. Файлы, перечисленные в `.dockerignore`, не попадут в контекст сборки (и в образ через `COPY . .`).

Создайте файл `.dockerignore` в корне проекта:

```
.git
.gitignore
.idea
*.iml
target
node_modules
*.md
docker-compose*.yml
compose.yaml
Dockerfile
.env
```

Без `.dockerignore` при каждом `docker build` в контекст сборки уходит папка `.git` (может весить сотни мегабайт), `target`, `node_modules` и т.д. Это замедляет сборку и засоряет кэш.

### 2. Минимизируйте количество слоёв

Каждая инструкция `RUN` создаёт новый слой. Объединяйте связанные команды:

```dockerfile
# ❌ Плохо — 3 слоя
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Хорошо — 1 слой
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Используйте кэширование слоёв

Docker кэширует слои. Если файл не изменился — слой берётся из кэша. Располагайте инструкции от редко меняющихся к часто меняющимся:

```dockerfile
# ✅ Правильный порядок для Java/Maven
COPY pom.xml .
RUN mvn dependency:go-offline    # Кэшируется, пока pom.xml не изменится
COPY src ./src                   # Исходники меняются часто — этот слой пересобирается
RUN mvn package -DskipTests
```

### 4. Не запускайте контейнеры от root

По умолчанию процесс внутри контейнера работает от root. Это небезопасно. Создайте непривилегированного пользователя:

```dockerfile
FROM eclipse-temurin:21-jre
RUN groupadd -r appuser && useradd -r -g appuser appuser
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN chown -R appuser:appuser /app
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5. Используйте health checks

В `compose.yaml`:

```yaml
services:
  app:
    # ...
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

Для Spring Boot добавьте зависимость `spring-boot-starter-actuator` — она предоставляет эндпоинт `/actuator/health`.

В Dockerfile:

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=40s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
```

### 6. Указывайте конкретные теги

```dockerfile
# ❌ Плохо — latest может сломаться в любой момент
FROM eclipse-temurin:latest

# ⚠️ Нормально — мажорная версия
FROM eclipse-temurin:21-jre

# ✅ Отлично — полная версия
FROM eclipse-temurin:21.0.6_7-jre
```

### 7. Один процесс — один контейнер

Не запускайте в одном контейнере и приложение, и БД, и nginx. Каждый сервис — отдельный контейнер. Compose объединяет их в систему.

### 8. Используйте переменные окружения для конфигурации

Не хардкодьте пароли, URL базы и прочие настройки в `application.properties`. Передавайте через `environment` в Compose:

```yaml
environment:
  SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/tracker
  SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}  # из .env файла
```

Файл `.env` (рядом с `compose.yaml`):

```
DB_PASSWORD=mysecretpassword
```

> `.env` **не коммитьте** в Git! Добавьте его в `.gitignore`.

### 9. Полезные команды для отладки

```bash
# Зайти внутрь работающего контейнера
docker exec -it <container> bash

# Посмотреть логи
docker logs <container>
docker logs -f --tail 100 <container>

# Статистика ресурсов (CPU, RAM)
docker stats

# Посмотреть, из чего состоит образ (по слоям)
docker history <image>

# Полная очистка: контейнеры, образы, тома, сети
docker system prune -a --volumes
```

### 10. Итоговый чек-лист

- [ ] `.dockerignore` создан
- [ ] Multi-stage build: сборка и запуск разделены
- [ ] Используется JRE (не JDK) в runtime-этапе
- [ ] Контейнер не запускается от root
- [ ] Теги образов конкретные (не `latest`)
- [ ] `compose.yaml` без поля `version`
- [ ] Health check настроен
- [ ] Секреты не захардкожены (используется `.env` + `.gitignore`)

### Задание

1. Возьмите любой ваш Docker-проект из предыдущих уроков.
2. Пройдите по чек-листу выше и примените все рекомендации, которые ещё не применены.
3. Создайте `.dockerignore`.
4. Добавьте health check.
5. Убедитесь, что контейнер работает не от root.
6. Оставьте ссылку на коммит.
