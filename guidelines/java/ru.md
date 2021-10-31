# Java: Гайд по коду

## Общие положения

- `package-private` не допускается ни в каких проявлениях — использование его указывает на неграмотную инкапсуляцию и потенциальные неявные действия с классом.

- Любой публичный не-`static` метод должен быть описан интерфейсом, даже value-класс (последнее может казаться избыточным, однако таким образом не приходится искать эфимерную грань между value-классом и классом с логикой).
  Иключение составляют автоматически генерируемые методы `record`-классов.

- Всякий класс должен быть, по умолчанию, `final`, если только другое явно не требуется, в случае чего все его члены должны быть `protected`.

- Любые поля `final`-классов должны быть `private`, если же класс не `final`, то его поля должны быть `protected`.

  исключение составляют `static final` поля примитивных и иммутабельных типов (например, `java.lang.String`).

## Именование

Именование должно выполняться в соответствии со стандартным соглашением об именовании в Java. Однобуквенные именования переменных запрещаются, за исключением:

- неиспользуемых счётчиков в циклах, например `for (var i = 0; i < attemptsCount; i++)`
- пойманных исключений:
    - `e` для всякого типа исключения, наследующего (и, в том числе) `java.lang.Exception`
    - `x` для всякого типа исключения, не наследующего `java.lang.Exception`

Аббревиатуры сокращаются как обычные слова, например `SqlHelper`, а не `SQLHelper`.

Сокращения не допускаются, за исключением общепринятых (`id`, а не `identifier`).

## Методы

- Параметры методов должны обладать наиболее общим типом.
- Параметры не-`abstract` методов, а так же локальные переменные должны помечаться `final`, если иное явно не требуется.
- Не допускается использование `synchronized` методов.

## Иммутабельность

Всегда, когда это возможно, классы должны быть настолько иммутабельными, насколько это возможно.

Поощряется использование `record`-классов.

Это даёт возможность более грамотно строить API взаимодействия, а в некоторых случаях также позволяет достигать более высокой производительности.

Таким образом:

- все поля по умолчанию `final`
- использование сеттеров противопоказано (за исключением тех случаев, когда это часть логики, описанной в интерфейсе)

Рекомендуется аннотировать класс `lombok.@FieldDefaults(level = AccessLevel.PRIVATE, makeFinal = true)`, что сделает его поля, по-умолчанию, `private final`.

Допускается и поощряется использование [**Immutables**](https://immutables.github.io/) там, где использования `record` невозможно или недостаточно.

## Фабрики против конструкторов

Публичные конструкторы находятся под запретом, как и конструкторы с любой логикой, кроме тривиальной инициализации `final` полей.

Вместо них нужно использовать статичные фабрики, возвращающие более слабый тип (а именно, интерфейс, реализация которого возвращается), размещая всякую логику в них.

Например вместо

```java
public final class MyContainer implements Container {
    private final @NotNull Foo foo;
    private final @NotNull Bar bar;

    public MyContainer(final @NotNull Foo foo, final @NotNull Bar bar) {
        if (foo == null) throw new NullPointerException("foo is null");
        if (bar == null) throw new NullPointerException("bar is null");

        this.foo = foo;
        this.bar = bar;
    }
}
```

Стоит использовать

```java
public final class MyContainer implements Container {
    private final @NotNull Foo foo;
    private final @NotNull Bar bar;

    private MyContainer(final @NotNull Foo foo, final @NotNull Bar bar) {
        this.foo = foo;
        this.bar = bar;
    }

    public static @NotNull Container create(final @NotNull Foo foo,
                                            final @NotNull Bar bar) {
        if (foo == null) throw new NullPointerException("foo is null");
        if (bar == null) throw new NullPointerException("bar is null");

        return new MyContainer(foo, bar);
    }
}
```

С использованием Lombok:

```java
@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
@FieldDefaults(level = AccessLevel.PRIVATE, makeFinal = true)
public final class MyContainer implements Container {
    @NotNull Foo foo;
    @NotNull Bar bar;

    public static @NotNull Container create(final @NonNull Foo foo, final @NonNull Bar bar) {
        return new MyContainer(foo, bar);
    }
}
```

## Result вместо исключений

Для обработки ошибок бизнес-логики рекомендуется использовать [`Result`](https://javadoc.io/doc/ru.progrm-jarvis/java-commons/latest/ru/progrm_jarvis/javacommons/object/Result.html). Это позволяет:

- избежать накладных расходов связанных с построением стек-трейса исключения, либо необходимости явно их отключать для данного типа исключения;
- статически гарантировать обработку ошибки в произвольной точке исполнения;
- более гибко управлять содержимым ошибочной ситуации;
- использовать в качестве типа ошибки произвоьную `sealed`-иерархию.

В случае необходимости обеспечить выбрасывание исключения допускается использование семейства методов `Result#(Error)?orElse(Sneaky)?Throw(..)`, `ResultExtensions#rethrow(..)`, а в случае необходимости перейти от исключения к `Result` - `Result#tryFrom(..)`, однако эти методы должны использоваться лишь для поддержки устаревших API.

## Lombok

Рекомендуется (хоть и не является обязательным) использование **[Lombok](https://projectlombok.org/)**. Все его аннотации (включая `val`) разрешены к использованию, если это не противоречит данному гайду, за исключением:

- `@Cleanup` — вместо неё нужно использовать конструкцию `try-with-resource`;

- `@Synchronized` — всякие блокировки должны быть явными;  в целом, синхронизация на основе захвата монитора, должна быть последней мерой в борьбе с проблемами многопоточности.

Рекомендуется по умолчанию использовать следующую пару аннотаций для любого не-утилитного класса:

```java
@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
@FieldDefaults(level = AccessLevel.PRIVATE, makeFinal = true)
```

## Аннотации типов

Все упоминания типов в исходном коде должны быть аннотированны соответствующими аннотациями типов:

### `null`-безопасность

- `org.jetbrains.annotations.NotNull` — ожидается не-`null` значение, передача `null` ведёт к неопределённому поведению (скорее всего, в какой-то момент вызовется `NullPointerException`, однако это не гарантированно, и рантайм нельзя считать корректным, если этот инвариант нарушен);
- `org.jetbrains.annotations.Nullable` — допускается `null` значение;
- `lombok.NonNull` — ожидается не-`null` значение, передача `null` ведёт к немедленному выбрасыванию `NullPointerException` (с учётом остальных правил, уместно только для параметров методов).

Пример:

```java
public interface Service<@NotNull K> {

    @Nullable V get(@NonNull K key);

    void set(@NonNull K key, @Nullable Object value);
}

public final class NaiveService<@NotNull K> implements Service<K> {

    @NotNull Map<@NotNull K, @Nullable Object> map;

    // Тривиальный приватный конструктор,
    // валидацию входных данных должен производить вызывающий его
    //
    // PS: эквивалентно @RequiredArgsConstructor(access = AccessLevel.PRIVATE)
    private NaiveService(final @NotNull Map<@NotNull K, @Nullable Object> map) {
        this.map = map;
    }

    // публичный factory-метод, на нём лежит ответственность за валидацию входных данных
    public static <@NotNull K> @NotNull Service<@NotNull K> create(
    final @NonNull Map<@NotNull K, Object>
    ) {
        return new NaiveService<>(map);
    }

    @Override
    public @Nullable V get(@NonNull K key) {
        return map.get(key);
    }

    @Override
    public void set(@NonNull K key, @Nullable Object value) {
        map.put(key, value);
    }
}
```

### Дополнительные контракты

- `org.jetbrains.annotations.Unmodifiable`:
    - Применительно к возвращаемым типам методов — гарантирует, что возвращаемая коллекция не может подвергаться изменениям (возвращающий обязан защитить её от модификация, например обернув в `Collections.unmodifiableList()`;
    - Применительно к не-`public` членам — запрещает их модификацию, но не требует рантайм-проверок (иными словами, попытка модифицировать её приводит к неопределённому поведению).
- `org.jetbrains.annotations.UnmodifiableView` — аналогично `org.jetbrains.annotations.Unmodifiable`, однако явно указывает на то, что аннотированный элемент является лишь отображением коллекции, а не ей самой.

## Другие аннотации

Рекомендуется использование аннотаций:

- `org.jetbrains.annotations.Contract` — рекомеднуется использовать для публичных методов, дабы более точно специфицировать их поведение.

В целом, любые аннотации из состава `org.jetbrains:annotations` допускаются к использованию, если они не противоречат этому гайду.

## Инспекции

Рекомендуется использовать набор инспекций **[CruelDefaults](https://www.notion.so/9ae9610adb67c79a449accd17516accf)**.

## Рекомендованные плагины для IntellijIDEA

Вспомогательные:

- [Maven Helper](https://plugins.jetbrains.com/plugin/7179-maven-helper)
- [GitToolBox](https://plugins.jetbrains.com/plugin/7499-gittoolbox)

Визуальные

- [Code Glance](https://plugins.jetbrains.com/plugin/7275-codeglance)
- [Rainbow Brackets](https://plugins.jetbrains.com/plugin/10080-rainbow-brackets)
- [Indent Rainbow](https://plugins.jetbrains.com/plugin/13308-indent-rainbow)
- [Extra Icons](https://plugins.jetbrains.com/plugin/11058-extra-icons)
- [Nyan Progress Bar](https://plugins.jetbrains.com/plugin/8575-nyan-progress-bar)

Рекомендованные:

- [Key Promoter X](https://plugins.jetbrains.com/plugin/9792-key-promoter-x)