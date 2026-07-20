# Factory Pattern — мастер-класс (Java + Spring)

## 1. Проблема, которую решает Factory

Представь задачу из банка: система должна отправлять уведомления клиентам —
EMAIL, SMS, PUSH. Наивное решение выглядит так:

```java
// ❌ АНТИПАТТЕРН: switch размазан по сервису
@Service
public class NotificationService {

    public void notify(Client client, String message, NotificationType type) {
        switch (type) {
            case EMAIL -> {
                // 30 строк логики отправки email
            }
            case SMS -> {
                // 30 строк логики отправки sms
            }
            case PUSH -> {
                // 30 строк логики push
            }
        }
    }
}
```

**Чем это плохо (это спросят на собесе!):**
1. **Нарушение OCP** (Open/Closed Principle): добавить WHATSAPP = лезть в этот класс и менять его.
2. **Нарушение SRP**: один класс знает, КАК отправлять всеми способами сразу.
3. Такой `switch` начнёт копироваться по проекту (валидация, логирование, ретраи — везде свой switch).
4. Невозможно нормально тестировать по отдельности.

Factory решает это: **логика выбора реализации выносится в одно место**,
а сами реализации изолированы друг от друга.

---

## 2. Классический Factory Method (GoF, без Spring)

Сначала база — чтобы ты мог рассказать теорию.

```java
// Продукт — общий интерфейс того, что создаём
public interface NotificationSender {
    void send(String recipient, String message);
}

// Конкретные продукты
public class EmailSender implements NotificationSender {
    @Override
    public void send(String recipient, String message) {
        System.out.println("EMAIL to " + recipient + ": " + message);
    }
}

public class SmsSender implements NotificationSender {
    @Override
    public void send(String recipient, String message) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

// ✅ Простая фабрика (Simple Factory) — статический метод создания.
// Важно: клиентский код теперь НЕ знает про конкретные классы,
// он знает только интерфейс NotificationSender.
public class NotificationSenderFactory {

    public static NotificationSender create(NotificationType type) {
        return switch (type) {
            case EMAIL -> new EmailSender();
            case SMS   -> new SmsSender();
            case PUSH  -> new PushSender();
        };
        // Да, switch остался. НО: он теперь ровно в ОДНОМ месте.
        // Это уже огромный выигрыш. В Spring мы уберём и его (см. ниже).
    }
}
```

### Терминология для собеседования (не путай!)

| Вариант | Что это |
|---|---|
| **Simple Factory** | Статический метод `create(type)` со switch. Формально даже не GoF-паттерн, а идиома. То, что выше. |
| **Factory Method (GoF)** | Абстрактный класс/интерфейс объявляет метод `createProduct()`, а **подклассы решают**, какой объект создать. Выбор через наследование. |
| **Abstract Factory (GoF)** | Фабрика, создающая **семейства** связанных продуктов (например, `KaspiUiFactory` создаёт и кнопку, и окно в одном стиле). |

Пример настоящего Factory Method (через наследование):

```java
// Создатель объявляет фабричный метод, но не знает конкретный продукт
public abstract class NotificationDispatcher {

    // Это и есть factory method — "дырка", которую заполняет подкласс
    protected abstract NotificationSender createSender();

    // Шаблонная логика работает с абстракцией
    public void dispatch(String recipient, String message) {
        NotificationSender sender = createSender(); // подкласс решит, кто это
        sender.send(recipient, message);
    }
}

public class EmailDispatcher extends NotificationDispatcher {
    @Override
    protected NotificationSender createSender() {
        return new EmailSender(); // выбор сделан наследованием, без switch
    }
}
```

Заметь: Factory Method часто живёт в паре с **Template Method** —
`dispatch()` здесь и есть шаблонный метод. Это хороший мостик на собесе.

---

## 3. 🏆 Золотой стандарт: Factory в Spring

В Spring мы не пишем `new` и не пишем switch. Вместо этого используем
**инъекцию всех реализаций интерфейса** — Spring сам находит все бины
типа `NotificationSender` и отдаёт их списком.

### Шаг 1. Интерфейс с методом самоидентификации

```java
public interface NotificationSender {

    void send(String recipient, String message);

    // 🔑 Ключевая идея: каждая реализация САМА говорит, какой тип она обслуживает.
    // Благодаря этому фабрика не знает конкретных классов вообще.
    NotificationType getType();
}
```

### Шаг 2. Реализации — обычные Spring-бины

```java
@Component
public class EmailSender implements NotificationSender {

    private final JavaMailSender mailSender; // реальные зависимости инжектятся как обычно

    public EmailSender(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    @Override
    public void send(String recipient, String message) {
        // логика email
    }

    @Override
    public NotificationType getType() {
        return NotificationType.EMAIL;
    }
}

@Component
public class SmsSender implements NotificationSender {

    @Override
    public void send(String recipient, String message) {
        // логика sms
    }

    @Override
    public NotificationType getType() {
        return NotificationType.SMS;
    }
}
```

### Шаг 3. Фабрика: List → Map

```java
@Component
public class NotificationSenderFactory {

    // Map: тип уведомления -> его обработчик
    private final Map<NotificationType, NotificationSender> senders;

    // 🔑 МАГИЯ SPRING: если попросить List<NotificationSender>,
    // Spring соберёт в него ВСЕ бины, реализующие этот интерфейс.
    // Добавил новый @Component PushSender — он попадёт сюда АВТОМАТИЧЕСКИ.
    public NotificationSenderFactory(List<NotificationSender> senderList) {
        this.senders = senderList.stream()
                .collect(Collectors.toUnmodifiableMap(
                        NotificationSender::getType,   // ключ — тип
                        Function.identity()            // значение — сам бин
                ));
        // toUnmodifiableMap — фабрика иммутабельна после создания (потокобезопасно).
        // Бонус: если два бина вернут один и тот же getType(),
        // toUnmodifiableMap кинет IllegalStateException НА СТАРТЕ приложения —
        // fail-fast, ошибку конфигурации поймаем сразу, а не в проде.
    }

    public NotificationSender getSender(NotificationType type) {
        NotificationSender sender = senders.get(type);
        if (sender == null) {
            // Явное сообщение лучше, чем NPE где-то дальше
            throw new IllegalArgumentException(
                    "No sender registered for type: " + type);
        }
        return sender;
    }
}
```

### Шаг 4. Клиентский код — чистый

```java
@Service
public class NotificationService {

    private final NotificationSenderFactory factory;

    public NotificationService(NotificationSenderFactory factory) {
        this.factory = factory;
    }

    public void notify(Client client, String message, NotificationType type) {
        // Никаких if/switch. Никакого знания о конкретных классах.
        factory.getSender(type).send(client.getContact(type), message);
    }
}
```

### Что получили

1. **OCP соблюдён идеально**: новый канал = новый класс `@Component WhatsAppSender`.
   Ни фабрику, ни сервис трогать НЕ надо.
2. **SRP**: каждый sender отвечает только за свой канал.
3. Каждый sender тестируется изолированно, у каждого свои зависимости.
4. Выбор реализации — O(1) по Map.

---

## 4. Вариации, о которых могут спросить

### 4.1. Инъекция Map напрямую

```java
// Spring умеет инжектить Map<String, NotificationSender>,
// где ключ — ИМЯ БИНА ("emailSender", "smsSender").
public NotificationSenderFactory(Map<String, NotificationSender> sendersByBeanName) { ... }
```
Работает, но ключ — строка с именем бина. Хрупко: переименовал класс —
сломал логику. Вариант с `getType()` + enum надёжнее, его и называй основным.

### 4.2. EnumMap как оптимизация

```java
this.senders = new EnumMap<>(
        senderList.stream().collect(Collectors.toMap(
                NotificationSender::getType, Function.identity())));
```
`EnumMap` — массив под капотом, быстрее и компактнее HashMap для enum-ключей.
Приятная деталь, чтобы блеснуть.

### 4.3. Где Factory в самом Spring / JDK? (любимый вопрос)

- `BeanFactory` — сам Spring-контейнер это гигантская фабрика бинов.
- `@Bean`-методы в `@Configuration` — фабричные методы.
- `FactoryBean<T>` — интерфейс Spring для кастомного создания сложных бинов.
- JDK: `List.of(...)`, `Optional.of(...)`, `Executors.newFixedThreadPool(...)`,
  `LocalDate.now()` — статические фабричные методы (Effective Java, Item 1).

### 4.4. Factory vs Strategy — в чём разница? (вопрос-ловушка)

Структурно код почти одинаковый! Разница в **намерении**:
- **Factory** отвечает на вопрос «**какой объект создать/выдать**».
- **Strategy** — «**какой алгоритм применить**» к уже существующему контексту.

Наш Spring-пример — это, строго говоря, гибрид: Map хранит стратегии,
а фабрика их выдаёт. На собесе так и скажи — это покажет глубину понимания.

---

## 5. Чек-лист для самопроверки

- [ ] Могу объяснить, чем плох switch в сервисе (OCP, SRP)
- [ ] Различаю Simple Factory / Factory Method / Abstract Factory
- [ ] Могу написать Spring-фабрику через List → Map по памяти
- [ ] Знаю, зачем toUnmodifiableMap и что будет при дубликате ключа
- [ ] Могу назвать примеры фабрик в Spring и JDK
- [ ] Могу объяснить разницу Factory vs Strategy
