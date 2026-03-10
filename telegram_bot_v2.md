# Создание Telegram-бота на Spring Boot с обработкой команд и пользовательского ввода [v2, 2026]

## Введение

Цель этого урока — показать, как на Java с использованием Spring Boot создавать Telegram-ботов с чистой архитектурой. Мы научим бота «запоминать» контекст общения с пользователем, при этом избежим огромных `if-else` блоков.

> **Что изменилось по сравнению с оригинальным уроком:**
> - Оригинал использовал «голую» библиотеку TelegramBots с наследованием от `TelegramLongPollingBot` и ручной регистрацией бота через `TelegramBotsApi`.
> - В 2026 году библиотека полностью перешла на Spring Boot Starter: бот — это обычный Spring-компонент (`@Component`), регистрация автоматическая, вместо наследования — реализация интерфейсов `SpringLongPollingBot` + `LongPollingSingleThreadUpdateConsumer`.
> - Вместо вызова `execute()` через наследование используется `TelegramClient` — отдельный бин, инъектируемый через конструктор.
> - Все настройки — через `application.properties`, без хардкода токена.

---

## Шаг 1: Регистрация бота в Telegram

Этот шаг не изменился:

1. Найдите бота **@BotFather** в Telegram.
2. Отправьте команду `/newbot`.
3. Укажите отображаемое имя и username (должен заканчиваться на `bot`).
4. Сохраните полученный **токен** — он понадобится для `application.properties`.

---

## Шаг 2: Создание проекта Spring Boot

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.4.1</version>
        <relativePath/>
    </parent>
    <groupId>ru.job4j</groupId>
    <artifactId>telegram-bot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <properties>
        <java.version>21</java.version>
    </properties>
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Telegram Bots: Spring Boot Long Polling Starter -->
        <dependency>
            <groupId>org.telegram</groupId>
            <artifactId>telegrambots-springboot-longpolling-starter</artifactId>
            <version>9.3.0</version>
        </dependency>

        <!-- Telegram Bots: OkHttp Client -->
        <dependency>
            <groupId>org.telegram</groupId>
            <artifactId>telegrambots-client</artifactId>
            <version>9.3.0</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

> **Было (старый урок):** одна зависимость `telegrambots` или `telegrambots-spring-boot-starter` версии 5.x–6.x.
>
> **Стало:** две зависимости — `telegrambots-springboot-longpolling-starter` (автоконфигурация Spring Boot) и `telegrambots-client` (OkHttp-клиент для отправки запросов к Telegram API). Версия 9.3.0 — актуальная на 2026 год.

### application.properties

```properties
telegram.bot.token=${TELEGRAM_BOT_TOKEN:your-token-here}
```

> Токен вынесен в properties. В продакшене передавайте его через переменную окружения `TELEGRAM_BOT_TOKEN`, а не храните в коде.

---

## Шаг 3: Что изменилось в базовом классе бота

В оригинальном уроке класс бота выглядел примерно так:

**Было (старый подход):**

```java
// ❌ Устаревший подход — НЕ используйте
public class MyBot extends TelegramLongPollingBot {

    @Override
    public String getBotUsername() {
        return "my_bot";          // хардкод
    }

    @Override
    public String getBotToken() {
        return "123456:ABC-DEF";  // хардкод токена!
    }

    @Override
    public void onUpdateReceived(Update update) {
        if (update.hasMessage() && update.getMessage().hasText()) {
            var chatId = update.getMessage().getChatId().toString();
            var text = update.getMessage().getText();
            execute(new SendMessage(chatId, "Эхо: " + text)); // вызов через наследование
        }
    }
}
```

Проблемы этого подхода:
- Токен и имя бота «зашиты» прямо в код
- `execute()` вызывается через наследование — сложно тестировать
- Бот не является Spring-компонентом — его нужно вручную регистрировать

**Стало (Spring Boot 2026):**

```java
// ✅ Современный подход
@Component
public class MyBot implements SpringLongPollingBot,
                              LongPollingSingleThreadUpdateConsumer {

    private final TelegramClient telegramClient;
    private final String token;

    public MyBot(@Value("${telegram.bot.token}") String token) {
        this.token = token;
        this.telegramClient = new OkHttpTelegramClient(token);
    }

    @Override
    public String getBotToken() {
        return token;
    }

    @Override
    public LongPollingUpdateConsumer getUpdatesConsumer() {
        return this;
    }

    @Override
    public void consume(Update update) {
        if (update.hasMessage() && update.getMessage().hasText()) {
            var chatId = update.getMessage().getChatId().toString();
            var text = update.getMessage().getText();
            try {
                telegramClient.execute(
                        new SendMessage(chatId, "Эхо: " + text)
                );
            } catch (TelegramApiException e) {
                e.printStackTrace();
            }
        }
    }
}
```

Разница:
- `@Component` — бот является Spring-бином, регистрация автоматическая
- Два интерфейса вместо наследования: `SpringLongPollingBot` (для интеграции со Spring) и `LongPollingSingleThreadUpdateConsumer` (для получения обновлений)
- `TelegramClient` — отдельный объект для отправки сообщений, вместо унаследованного `execute()`
- Токен инъектируется через `@Value`
- Точка входа — метод `consume(Update)` вместо `onUpdateReceived(Update)`

### Почему в примере echo-бот?

Оба примера выше — это echo-бот: бот получает сообщение и отправляет его обратно. Как эхо в горах — крикнул «привет», получил «привет» в ответ. Это стандартный «hello world» в мире Telegram-ботов: если эхо работает — значит токен правильный, сообщения приходят и уходят.

`SendMessage` — это объект, описывающий одно исходящее текстовое сообщение. Конструктор принимает два параметра: кому (`chatId`) и что (`text`). Вызов `telegramClient.execute(sendMessage)` — это HTTP-запрос к Telegram API, который доставляет сообщение пользователю.

Но `consume()` — это точка входа для **всех** обновлений от Telegram, а не только текстовых. Вот что ещё можно делать внутри:

- **Отправлять кнопки.** `SendMessage` можно снабдить `ReplyKeyboardMarkup` (кнопки внизу экрана) или `InlineKeyboardMarkup` (кнопки прямо в сообщении).
- **Отправлять фото, документы, стикеры.** Вместо `SendMessage` используются `SendPhoto`, `SendDocument`, `SendSticker` — всё тот же `telegramClient.execute()`, просто другой объект.
- **Обрабатывать callback от inline-кнопок.** Когда пользователь нажимает inline-кнопку, приходит не `message`, а `callbackQuery`. Его тоже проверяют в `consume()` через `update.hasCallbackQuery()`.
- **Маршрутизировать команды.** Именно этим мы займёмся дальше — вместо `if-else` на каждую команду вынесем логику в отдельные классы `Action`.

В нашем уроке echo-бот нужен ровно для одной цели — показать минимальный рабочий пример в блоке «было/стало», чтобы была видна разница между `this.execute()` и `telegramClient.execute()`. Дальше мы строим бота по-настоящему — через `Action` и карту команд.

---

## Шаг 3.1: Разбираем интерфейсы — что зачем

Может показаться странным: зачем два интерфейса, если бот — это «один класс»? Давайте разберёмся, какую роль играет каждый из них и какие есть альтернативы.

### SpringLongPollingBot — интеграция со Spring

Этот интерфейс нужен **только** для автоматической регистрации бота через Spring Boot Starter. Его контракт минимален:

```java
public interface SpringLongPollingBot {
    String getBotToken();
    LongPollingUpdateConsumer getUpdatesConsumer();
}
```

«Дай мне токен и скажи, кто будет обрабатывать обновления» — вот и весь контракт. При старте приложения Spring Boot Starter находит все бины типа `SpringLongPollingBot` и регистрирует их автоматически.

Альтернативы у этого интерфейса нет — если хочешь автоматическую регистрацию через Spring, реализуешь `SpringLongPollingBot`. Если Spring не используешь — регистрируешь бота вручную через `TelegramBotsLongPollingApplication.registerBot(token, consumer)`, и тогда `SpringLongPollingBot` не нужен вообще.

### LongPollingUpdateConsumer — как получать обновления

`getUpdatesConsumer()` должен вернуть `LongPollingUpdateConsumer`. Это базовый интерфейс с одним методом:

```java
public interface LongPollingUpdateConsumer {
    void consume(List<Update> updates);  // получает ПАЧКУ обновлений
}
```

Telegram присылает обновления пачками (до 100 штук за один запрос). Базовый интерфейс отдаёт весь список. И вот тут развилка:

### Вариант 1: LongPollingSingleThreadUpdateConsumer (простой)

Это **утилитный интерфейс**, который разворачивает список за вас:

```java
public interface LongPollingSingleThreadUpdateConsumer
        extends LongPollingUpdateConsumer {

    void consume(Update update);  // ОДНО обновление

    @Override
    default void consume(List<Update> updates) {
        updates.forEach(this::consume);  // просто цикл
    }
}
```

Он реализует `consume(List<Update>)` за вас — проходит по списку и для каждого элемента вызывает `consume(Update)` с одним обновлением. Всё последовательно, в одном потоке. Это самый простой вариант, и он подходит для подавляющего большинства ботов.

**Когда использовать:** обычный бот, учебный проект, небольшая нагрузка. Порядок обработки сообщений гарантирован.

### Вариант 2: LongPollingUpdateConsumer напрямую (продвинутый)

Если вам нужна параллельная обработка — реализуйте базовый интерфейс напрямую:

```java
@Component
public class HighLoadBot implements SpringLongPollingBot,
                                    LongPollingUpdateConsumer {

    private final ExecutorService executor =
            Executors.newVirtualThreadPerTaskExecutor();
    private final TelegramClient telegramClient;
    private final String token;

    public HighLoadBot(@Value("${telegram.bot.token}") String token) {
        this.token = token;
        this.telegramClient = new OkHttpTelegramClient(token);
    }

    @Override
    public String getBotToken() {
        return token;
    }

    @Override
    public LongPollingUpdateConsumer getUpdatesConsumer() {
        return this;
    }

    @Override
    public void consume(List<Update> updates) {
        for (Update update : updates) {
            executor.submit(() -> processUpdate(update));
        }
    }

    private void processUpdate(Update update) {
        // обработка в отдельном виртуальном потоке
    }
}
```

Здесь каждое обновление обрабатывается в отдельном виртуальном потоке (Java 21+). Вы сами управляете параллельностью и порядком обработки.

**Когда использовать:** высоконагруженный бот с тысячами сообщений в секунду, нужна параллельная обработка, группировка или фильтрация пачкой.

### Итого по интерфейсам

```
SpringLongPollingBot                    ← «зарегистрируй меня в Spring»
  └─ getBotToken()
  └─ getUpdatesConsumer() → возвращает:
        │
        ├─ LongPollingUpdateConsumer           ← consume(List<Update>)
        │    пачка обновлений, сам решаешь     ← для продвинутых
        │
        └─ LongPollingSingleThreadUpdateConsumer ← consume(Update)
             по одному, последовательно         ← для большинства ботов ✅
```

В нашем уроке мы используем `LongPollingSingleThreadUpdateConsumer`, потому что для учебного бота с картой команд это идеальный выбор — обновления приходят по одному, обрабатываются последовательно, никаких гонок данных.

---

## Шаг 3.2: Бот = один класс. Остальное — ваша архитектура

Важно понять: **с точки зрения библиотеки TelegramBots, бот — это один-единственный класс**, который реализует два интерфейса и три метода (`getBotToken`, `getUpdatesConsumer`, `consume`). Всё.

Всё остальное — `Action`, карты команд, `bindingBy`, сервисы, базы данных — это уже **ваша** архитектура приложения. Библиотеке до этого дела нет. Она просто вызывает `consume()` каждый раз, когда приходит обновление от Telegram, а что вы внутри делаете — ваше решение.

Аналогия со Spring MVC: `DispatcherServlet` — это тоже «один класс», точка входа для всех HTTP-запросов. А контроллеры, сервисы, репозитории — детали приложения. Наш `TrackerBot.consume()` играет ту же роль — принимает входящее событие и маршрутизирует его дальше.

---

## Шаг 3.3: Меню команд в Telegram

Вы наверняка видели у ботов кнопку **«/»** (или иконку меню) рядом с полем ввода — при нажатии выпадает список доступных команд. Это не магия — это нужно настроить. Есть два способа.

### Способ 1: Через BotFather (вручную)

Отправьте `/setcommands` в чат с @BotFather, выберите бота и введите список:

```
start - Начало работы
help - Список команд
new - Регистрация пользователя
```

Просто и быстро, но есть минус: при добавлении новой команды в код нужно не забыть обновить список и в BotFather. Забудете — пользователи не увидят новую команду в меню.

### Способ 2: Через код (программно) ✅

Правильнее регистрировать команды при старте бота — тогда меню обновляется автоматически при каждом деплое:

```java
import org.telegram.telegrambots.longpolling.BotSession;
import org.telegram.telegrambots.longpolling.starter.AfterBotRegistration;
import org.telegram.telegrambots.meta.api.methods.commands.SetMyCommands;
import org.telegram.telegrambots.meta.api.objects.commands.BotCommand;
import org.telegram.telegrambots.meta.api.objects.commands.scope.BotCommandScopeDefault;
```

```java
@AfterBotRegistration
public void afterRegistration(BotSession botSession) {
    try {
        telegramClient.execute(new SetMyCommands(
                List.of(
                        new BotCommand("/start", "Начало работы"),
                        new BotCommand("/help", "Список команд"),
                        new BotCommand("new", "Регистрация пользователя")
                ),
                new BotCommandScopeDefault(),
                null  // languageCode — null означает «для всех языков»
        ));
        log.info("Команды меню установлены");
    } catch (TelegramApiException e) {
        log.error("Ошибка установки команд меню", e);
    }
}
```

Этот метод добавляется прямо в класс бота (`TrackerBot`). Разберём по частям:

- `@AfterBotRegistration` — аннотация из Spring Boot Starter TelegramBots. Метод вызывается **один раз**, сразу после того, как бот зарегистрирован и сессия с Telegram активна. Это гарантия, что `telegramClient` уже работает.
- `SetMyCommands` — запрос к Telegram API, который устанавливает список команд для меню.
- `BotCommand("/start", "Начало работы")` — одна команда: первый аргумент — текст команды, второй — описание, которое видит пользователь в меню.
- `BotCommandScopeDefault()` — команды видны **всем** пользователям во **всех** чатах. Можно настроить и точнее: `BotCommandScopeChat` — для конкретного чата, `BotCommandScopeAllGroupChats` — только для групповых чатов, и т.д.
- `null` (languageCode) — команды одинаковые для всех языков. Можно вызвать `SetMyCommands` несколько раз с разными `languageCode`, чтобы русскоязычные пользователи видели описания на русском, а англоязычные — на английском.

> В нашем уроке мы используем второй способ. Полный код `TrackerBot` с меню будет показан в Шаге 5.

---

## Шаг 4: Проектирование архитектуры (Интерфейс Action)

Архитектурная идея оригинального урока остаётся актуальной и в 2026 году — она не зависит от версии библиотеки. Суть: чтобы метод обработки обновлений не превратился в бесконечную простыню `if-else`, мы выносим каждую команду в отдельный класс.

### Интерфейс Action

```java
package ru.job4j.bot.action;

import org.telegram.telegrambots.meta.api.objects.message.Message;
import org.telegram.telegrambots.meta.generics.TelegramClient;

public interface Action {

    /**
     * Имя команды, по которому Action регистрируется в карте.
     * Например: "/start", "New", "/help"
     */
    String commandName();

    /**
     * Описание команды для меню Telegram.
     * Пользователь увидит его рядом с командой при нажатии на кнопку «/».
     */
    String description();

    /**
     * Вызывается, когда пользователь выбирает команду
     * (например, нажимает кнопку или вводит "/start").
     */
    void handle(Message message, TelegramClient client);

    /**
     * Вызывается, когда пользователь вводит данные
     * в ответ на инструкцию бота.
     */
    void callback(Message message, TelegramClient client);
}
```

Четыре метода — четыре ответственности:
- `commandName()` — **идентификация**: как бот найдёт этот Action в карте команд. Именно по этому значению `TrackerBot` автоматически строит `Map<String, Action>` из всех Spring-бинов.
- `description()` — **описание для меню**: текст, который пользователь видит в меню команд Telegram. Используется в `@AfterBotRegistration` для автоматической регистрации меню.
- `handle()` — **реакция на команду**: пользователь ввёл `/start` или нажал кнопку «New».
- `callback()` — **реакция на ввод**: пользователь ответил на вопрос бота (например, ввёл email после команды «New»).

> В оригинальном уроке интерфейс `Action` имел только `handle()` и `callback()`, а карта команд заполнялась вручную. Мы добавили `commandName()`, чтобы Spring DI мог автоматически собрать карту из всех бинов `Action` — без ручной регистрации. `TelegramClient` передаётся как параметр вместо доступа через наследование — это делает `Action` легко тестируемым: в тестах можно подставить мок.

### Две карты

Бот хранит две ключевые структуры данных:

```
┌─────────────────────────────────────────────────┐
│  actions: Map<String, Action>                   │
│  ─────────────────────────────────               │
│  "/start"  → StartAction                        │
│  "New"     → RegAction                          │
│  "/help"   → HelpAction                         │
│                                                 │
│  Связь: текст команды → объект-обработчик       │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  bindingBy: Map<Long, Action>                   │
│  ─────────────────────────────────               │
│  123456789 → RegAction                          │
│  987654321 → (пусто)                            │
│                                                 │
│  Связь: chatId пользователя → действие,         │
│  которое ожидает от него ввода данных           │
└─────────────────────────────────────────────────┘
```

---

## Шаг 5: Логика обработки сообщений

```
Пользователь отправляет сообщение
           │
           ▼
┌─── Текст есть в карте actions? ───┐
│                                    │
│  ДА                                │  НЕТ
│                                    │
▼                                    ▼
Найти Action                  Есть ли chatId в bindingBy?
Вызвать action.handle()              │
Записать в bindingBy:          ДА    │    НЕТ
  chatId → action                    │
                                ▼         ▼
                           Вызвать    Ответить:
                           action     "Неизвестная
                           .callback()  команда"
                           Удалить
                           из bindingBy
```

### Реализация в коде бота

```java
package ru.job4j.bot;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import org.telegram.telegrambots.client.okhttp.OkHttpTelegramClient;
import org.telegram.telegrambots.longpolling.BotSession;
import org.telegram.telegrambots.longpolling.interfaces.LongPollingUpdateConsumer;
import org.telegram.telegrambots.longpolling.starter.AfterBotRegistration;
import org.telegram.telegrambots.longpolling.starter.SpringLongPollingBot;
import org.telegram.telegrambots.longpolling.util.LongPollingSingleThreadUpdateConsumer;
import org.telegram.telegrambots.meta.api.methods.commands.SetMyCommands;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.Update;
import org.telegram.telegrambots.meta.api.objects.commands.BotCommand;
import org.telegram.telegrambots.meta.api.objects.commands.scope.BotCommandScopeDefault;
import org.telegram.telegrambots.meta.exceptions.TelegramApiException;
import org.telegram.telegrambots.meta.generics.TelegramClient;
import ru.job4j.bot.action.Action;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Function;
import java.util.stream.Collectors;

@Slf4j
@Component
public class TrackerBot implements SpringLongPollingBot,
                                   LongPollingSingleThreadUpdateConsumer {

    private final TelegramClient telegramClient;
    private final String token;

    /**
     * Карта: текст команды → Action.
     * Заполняется автоматически из всех бинов Action в контексте Spring.
     */
    private final Map<String, Action> actions;

    /**
     * Карта: chatId → Action, который ожидает ввода от пользователя.
     * ConcurrentHashMap, т.к. бот может обрабатывать сообщения
     * от разных пользователей.
     */
    private final Map<Long, Action> bindingBy = new ConcurrentHashMap<>();

    public TrackerBot(@Value("${telegram.bot.token}") String token,
                      List<Action> actionBeans) {
        this.token = token;
        this.telegramClient = new OkHttpTelegramClient(token);
        this.actions = actionBeans.stream()
                .collect(Collectors.toMap(
                        Action::commandName,
                        Function.identity()
                ));
    }

    @Override
    public String getBotToken() {
        return token;
    }

    @Override
    public LongPollingUpdateConsumer getUpdatesConsumer() {
        return this;
    }

    @Override
    public void consume(Update update) {
        if (!update.hasMessage() || !update.getMessage().hasText()) {
            return;
        }
        var message = update.getMessage();
        var chatId = message.getChatId();
        var text = message.getText();

        var action = actions.get(text);
        if (action != null) {
            /* Пришла известная команда */
            action.handle(message, telegramClient);
            bindingBy.put(chatId, action);
        } else if (bindingBy.containsKey(chatId)) {
            /* Пришёл текст, и у пользователя есть незавершённое действие */
            var bound = bindingBy.remove(chatId);
            bound.callback(message, telegramClient);
        } else {
            /* Неизвестная команда */
            sendText(chatId, "Неизвестная команда. Используйте /start.");
        }
    }

    private void sendText(Long chatId, String text) {
        try {
            telegramClient.execute(
                    new SendMessage(chatId.toString(), text)
            );
        } catch (TelegramApiException e) {
            log.error("Ошибка отправки сообщения в чат {}", chatId, e);
        }
    }

    /**
     * Устанавливает меню команд в Telegram.
     * Вызывается один раз после успешной регистрации бота.
     */
    @AfterBotRegistration
    public void afterRegistration(BotSession botSession) {
        try {
            var commands = actions.values().stream()
                    .map(a -> new BotCommand(a.commandName(), a.description()))
                    .toList();
            telegramClient.execute(new SetMyCommands(
                    commands, new BotCommandScopeDefault(), null
            ));
            log.info("Команды меню установлены: {}", commands.size());
        } catch (TelegramApiException e) {
            log.error("Ошибка установки команд меню", e);
        }
    }
}
```

> **Ключевые отличия от оригинала:**
> - `consume()` вместо `onUpdateReceived()`.
> - `telegramClient.execute()` вместо `this.execute()` — клиент передаётся как зависимость.
> - Карта `actions` заполняется автоматически через Spring DI: все бины типа `Action` инъектируются как `List<Action>`.
> - `ConcurrentHashMap` для `bindingBy` — потокобезопасность.

---

## Шаг 6: Практический пример — Регистрация пользователя

### RegAction

```java
package ru.job4j.bot.action;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.message.Message;
import org.telegram.telegrambots.meta.exceptions.TelegramApiException;
import org.telegram.telegrambots.meta.generics.TelegramClient;

@Slf4j
@Component
public class RegAction implements Action {

    @Override
    public String commandName() {
        return "New";
    }

    @Override
    public String description() {
        return "Регистрация пользователя";
    }

    @Override
    public void handle(Message message, TelegramClient client) {
        send(client, message.getChatId(),
                "Введите почту для регистрации:");
    }

    @Override
    public void callback(Message message, TelegramClient client) {
        var email = message.getText();
        send(client, message.getChatId(),
                "Пользователь успешно зарегистрирован с почтой: " + email);
    }

    private void send(TelegramClient client, Long chatId, String text) {
        try {
            client.execute(new SendMessage(chatId.toString(), text));
        } catch (TelegramApiException e) {
            log.error("Ошибка отправки", e);
        }
    }
}
```

### StartAction

```java
package ru.job4j.bot.action;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.telegram.telegrambots.meta.api.methods.send.SendMessage;
import org.telegram.telegrambots.meta.api.objects.message.Message;
import org.telegram.telegrambots.meta.exceptions.TelegramApiException;
import org.telegram.telegrambots.meta.generics.TelegramClient;

@Slf4j
@Component
public class StartAction implements Action {

    @Override
    public String commandName() {
        return "/start";
    }

    @Override
    public String description() {
        return "Начало работы";
    }

    @Override
    public void handle(Message message, TelegramClient client) {
        var text = """
                Добро пожаловать! Доступные команды:
                /start — начало работы
                New — регистрация нового пользователя
                """;
        try {
            client.execute(
                    new SendMessage(message.getChatId().toString(), text)
            );
        } catch (TelegramApiException e) {
            log.error("Ошибка отправки", e);
        }
    }

    @Override
    public void callback(Message message, TelegramClient client) {
        /* /start не ожидает пользовательского ввода */
    }
}
```

### TelegramBotApplication

```java
package ru.job4j.bot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TelegramBotApplication {
    public static void main(String[] args) {
        SpringApplication.run(TelegramBotApplication.class, args);
    }
}
```

> Никакой ручной регистрации бота через `TelegramBotsApi` — стартер делает всё сам. Достаточно, чтобы в контексте Spring был бин, реализующий `SpringLongPollingBot`.

---

## Шаг 7: Демонстрация работы

```
Пользователь                    Бот
     │                           │
     │── /start ────────────────▶│
     │                           │  actions.get("/start") → StartAction
     │                           │  StartAction.handle()
     │◀── "Добро пожаловать!     │
     │    /start, New"           │
     │                           │
     │── New ───────────────────▶│
     │                           │  actions.get("New") → RegAction
     │                           │  RegAction.handle()
     │                           │  bindingBy.put(chatId, RegAction)
     │◀── "Введите почту         │
     │    для регистрации:"      │
     │                           │
     │── user@mail.ru ──────────▶│
     │                           │  actions.get("user@mail.ru") → null
     │                           │  bindingBy.get(chatId) → RegAction
     │                           │  RegAction.callback()
     │                           │  bindingBy.remove(chatId)
     │◀── "Пользователь          │
     │    зарегистрирован с      │
     │    почтой: user@mail.ru"  │
```

---

## Структура проекта

```
telegram-bot/
├── pom.xml
└── src/main/
    ├── java/ru/job4j/bot/
    │   ├── TelegramBotApplication.java
    │   ├── TrackerBot.java                 ← @Component, SpringLongPollingBot
    │   └── action/
    │       ├── Action.java                 ← интерфейс
    │       ├── StartAction.java            ← @Component
    │       └── RegAction.java              ← @Component
    └── resources/
        └── application.properties
```

---

## Сравнение: старый подход vs. новый

| Критерий | Старый (TelegramLongPollingBot) | Новый (Spring Boot Starter 9.x) |
|----------|-------------------------------|--------------------------------|
| **Базовый тип** | Наследование от `TelegramLongPollingBot` | Реализация `SpringLongPollingBot` + `LongPollingSingleThreadUpdateConsumer` |
| **Отправка сообщений** | `this.execute(...)` | `telegramClient.execute(...)` |
| **Регистрация** | Ручная через `TelegramBotsApi.registerBot()` | Автоматическая через Spring Boot Starter |
| **Токен** | `getBotToken()` с хардкодом | `@Value("${telegram.bot.token}")` |
| **Точка входа** | `onUpdateReceived(Update)` | `consume(Update)` |
| **Тестируемость** | Сложно — `execute()` через наследование | Просто — `TelegramClient` мокается |
| **DI для Action** | Ручное заполнение карты | Автоматическое через `List<Action>` |
| **Потокобезопасность** | Не учтена | `ConcurrentHashMap` для `bindingBy` |

---

## Как добавить новую команду

Чтобы добавить новую функцию боту, создайте новый класс:

```java
@Component
public class HelpAction implements Action {
    @Override
    public String commandName() { return "/help"; }

    @Override
    public String description() { return "Список команд"; }

    @Override
    public void handle(Message message, TelegramClient client) {
        // отправить справку
    }

    @Override
    public void callback(Message message, TelegramClient client) {
        // не используется
    }
}
```

Благодаря `@Component` и автоматическому сбору бинов в `TrackerBot`, новый класс подхватится без изменения кода бота. Это и есть принцип **Open-Closed** из SOLID — бот открыт для расширения, но закрыт для модификации.

---

## Ссылки

- [TelegramBots Documentation — Lesson 9: Spring Boot Bot](https://rubenlagus.github.io/TelegramBotsDocumentation/lesson-9.html)
- [TelegramBots GitHub](https://github.com/rubenlagus/TelegramBots)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [BotFather](https://t.me/BotFather)

---

## Заключение

Подход «один Action = один класс» из оригинального урока был отличной идеей — и она по-прежнему актуальна. Мы её сохранили, но модернизировали инфраструктуру: Spring Boot Starter автоматически регистрирует бота, `TelegramClient` инъектируется как зависимость вместо наследования, а Spring DI заполняет карту команд автоматически. Код стал проще, чище и легче тестируется.
