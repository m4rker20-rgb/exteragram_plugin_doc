# 10. Class Proxy: Java-подклассы из Python

Создание реальных Java-классов во время выполнения с реализацией на Python.

Class proxy API позволяет плагину создать настоящий Java-класс на лету и реализовать его в Python. Нужен, когда Java API ожидает:

- подкласс существующего Java-класса;
- реализацию с переопределением родительских методов;
- дополнительные методы, объявленные прямо в сгенерированном Java-классе;
- Java-поля на сгенерированном объекте.

Система построена на **DexMaker** — результат является настоящим Java-классом, который можно передавать куда угодно.

> ⚠️ `new_instance()` возвращает **Python-peer-объект**, а не сырой Java-инстанс. Используйте `.java`, когда API явно требует сам сгенерированный Java-объект.

## Рекомендуемый API — DSL из `extera_utils.classes`

```python
from extera_utils.classes import (
    Base, java_subclass, joverride, joverload, jmethod,
    jMVELmethod, jMVELoverride, jclassbuilder, jfield,
    jgetmethod, jsetmethod, jconstructor, jpreconstructor,
)
```

| Элемент | Назначение |
|---|---|
| `Base` | Базовый Python-класс для управляемых Java-подклассов |
| `@java_subclass(JavaClass, Interface1, ...)` | Привязка Python-класса к Java-базе и интерфейсам; опции `methods=`, `constructors=`, `custom_name=` |
| `@joverride(...)` | Переопределение существующего Java-метода |
| `@joverload(name, arg_types)` | Оверлоад-совместимый алиас `@joverride` |
| `@jmethod(...)` | Добавление нового метода в сгенерированный Java-класс |
| `jMVELmethod(...)` | Java-метод, реализованный на MVEL вместо Python |
| `jMVELoverride(...)` | Переопределение родительского Java-метода MVEL-кодом |
| `@jclassbuilder()` | Доступ к `DexMaker` до финализации и загрузки класса |
| `jfield(...)` | Добавление Java-поля |
| `jgetmethod(...)` / `jsetmethod(...)` | Java-only геттер/сеттер для поля |
| `@jconstructor(...)` | Python-код после завершения Java-конструктора |
| `@jpreconstructor(...)` | Модификация аргументов до вызова Java-конструктора |

## Быстрый старт

```python
from extera_utils.classes import Base, java_subclass, joverload, jfield
from java.util import ArrayList


@java_subclass(ArrayList)
class CountingList(Base):
    added_count = jfield("int", default=0)

    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.added_count += 1
        return super().add_item(value)


items = CountingList.new_instance()
items.add("Hello")
items.add("world")
```

Что произошло:

- `CountingList` стал сгенерированным Java-подклассом `java.util.ArrayList`;
- `added_count` — настоящее Java-поле этого класса;
- `add(Object)` переопределён в Java и маршрутизируется в Python;
- `super().add_item(...)` вызывает оригинальную Java-реализацию родителя.

Поскольку `ArrayList.add` перегружен, использован `@joverload(...)`. Для перегруженных Java-методов предпочитайте явные сигнатуры, а не «голый» `@joverride()`.

## Создание подкласса

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    ...
```

С интерфейсами:

```python
@java_subclass(FrameLayout, CustomDelegate1, CustomDelegate2)
class MyCell(Base):
    ...
```

Эквивалент: `class GeneratedProxy extends FrameLayout implements CustomDelegate1, CustomDelegate2`.

Привязка после определения класса:

```python
class MySubclass(Base):
    ...

MySubclass.bind(SomeJavaClass)
MySubclass.bind(FrameLayout, CustomDelegate1, CustomDelegate2)
```

Получить сгенерированный Java-класс: `MySubclass.java_class()`.

### Создание экземпляров

```python
instance = MySubclass.new_instance()
instance2 = MySubclass.new_instance("text", 123)      # аргументы Java-конструктора

java_instance = instance.java                          # сырой Java-объект
java_instance2 = MySubclass.new_java_instance("text", 123)
```

Отдельные аргументы для Python `__init__` — через `init_args`:

```python
instance = MySubclass.new_instance("java ctor value", init_args=["python init value"])
```

Восстановление peer-объекта из сырого Java-инстанса — `from_java(...)`:

```python
existing_java = MySubclass.new_java_instance("text")
peer = MySubclass.from_java(existing_java)
```

Полезно, когда Java API создал/сохранил прокси-инстанс за вас или вернул его через колбэк. Если peer существует — вернётся тот же Python-объект; если не существует — будет создан и привязан (кроме случая, когда объект был освобождён `jrelease()` — тогда вернётся `None`).

### Кастомное имя класса

Требуется version code **≥ 66690**:

```python
@java_subclass(Object, custom_name="DialogCell")
class Test(Base):
    ...
# → Proxy_Object_DialogCell_<hashcode>_<randomnum>
```

Без `custom_name` используется имя Python-класса: `Proxy_Object_Test_<number>_<number>`. Тоже доступно в `bind(...)`: `MySubclass.bind(SomeJavaClass, custom_name="SettingsRow")`. Имена санитизируются: неподдерживаемые символы → `_`.

Сгенерированные классы загружаются через общую цепочку загрузчиков, поэтому прокси, созданный позже, может ссылаться на более ранние прокси-классы. Но это не замена стабильному API-типу: предпочтительнее ссылаться на другой прокси через интерфейс или общий базовый класс.

## Переопределение методов

### Простое переопределение

Имена совпадают:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Clicked")
```

### С другим Python-именем

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride("equals")
    def equals_(self, a, b):
        return a == b
```

Полезно для `equals`, `hashCode` и когда нужно несколько Python-хендлеров для перегруженных методов.

### Перегруженные методы

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverload("setValue", ["int"])
    def set_value_int(self, value):
        print("int overload", value)

    @joverload("setValue", ["java.lang.String"])
    def set_value_text(self, value):
        print("string overload", value)
```

**Форматы типов:** примитивы (`"int"`, `"boolean"`, `"long"`), полные имена классов (`"java.lang.String"`), Java-классы из `jclass(...)`, объекты Java-примитивов:

```python
from java import jclass

View = jclass("android.view.View")

@joverload("setView", [View])
def set_view(self, view):
    ...
```

### Вызов родителя через `super()`

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onClick(self, view):
        print("Before parent")
        result = super().onClick(view)
        print("After parent")
        return result
```

Для перегруженных — вызывайте по своему Python-имени: `super().set_value_int(value)`.

## Добавление новых Java-методов: `@jmethod`

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jmethod("debugLabel", "java.lang.String", ["int"])
    def build_debug_label(self, value):
        return f"value={value}"
```

Создаёт настоящий Java-метод `debugLabel(int)`. Поддерживаемые формы:

```python
@jmethod("java.lang.String", ["int"])          # без имени метода (берётся из Python)
@jmethod("debugLabel", "java.lang.String", ["int"])  # с Java-именем
@jmethod()                                      # вывод из Python-аннотаций
def build_debug_label(self, value: "int") -> "java.lang.String":
    ...
```

## MVEL-методы для горячих путей

Если метод вызывается очень часто и логика простая — сгенерируйте метод, тело которого исполняет **MVEL** без захода в Python:

```python
from extera_utils.classes import Base, java_subclass, jMVELmethod


@java_subclass(SomeJavaClass)
class MySubclass(Base):
    debug_label = jMVELmethod(
        return_type="java.lang.String",
        arguments=[("prefix", "java.lang.String"), ("value", "int")],
        code="""
            return prefix + ": value=" + value + ", class=" + java.getClass().getSimpleName();
        """,
    )
```

Переопределение родителя MVEL-ом:

```python
@java_subclass(SomeFactoryClass)
class OptimizedFactory(Base):
    isClickable = jMVELoverride(
        arguments=[],
        code="return factory_is_clickable;",
    )
```

Для перегруженного родителя — явные типы, как в `@joverload(...)`:

```python
add = jMVELoverride(
    arguments=[("value", "java.lang.Object")],
    code="""
        add_calls = add_calls + 1;
        return SUPER_add(value);
    """,
)
```

### Контекст MVEL

- корневой объект — сгенерированный Java-инстанс (поля и методы доступны напрямую);
- `java` — тот же Java-инстанс;
- `python` / `py` / `self` — привязанный Python-peer;
- `args` — `Object[]` всех аргументов; `argc` — их количество; `arg0`, `arg1`, … — позиционные; плюс именованные из `arguments=[("name", "type"), ...]`;
- для override-методов доступен бридж `SUPER_<methodName>(...)`:

```python
onMeasure = jMVELoverride(
    arguments=[("widthMeasureSpec", "int"), ("heightMeasureSpec", "int")],
    code="""
        SUPER_onMeasure(
            android.view.View$MeasureSpec.makeMeasureSpec(
                android.view.View$MeasureSpec.getSize(widthMeasureSpec),
                android.view.View$MeasureSpec.EXACTLY
            ),
            android.view.View$MeasureSpec.makeMeasureSpec(
                org.telegram.messenger.AndroidUtilities.dp(68),
                android.view.View$MeasureSpec.EXACTLY
            )
        );
        return null;
    """,
)
```

### Когда использовать MVEL

1. `jfield(..., methods=[jgetmethod(...), jsetmethod(...)])` — самый быстрый вариант для чистых геттеров/сеттеров;
2. `jMVELmethod(...)` / `jMVELoverride(...)` — когда нужна небольшая логика без вызова Python;
3. держите MVEL-тела маленькими и предсказуемыми; для сложной логики с Python-объектами лучше обычные Python-переопределения.

## Java-поля: `jfield`

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    counter = jfield("int", default=0)
    title = jfield("java.lang.String", default="Hello")
```

Читаются и пишутся как обычные Python-атрибуты:

```python
self.counter += 1
self.title = "Updated"
```

### Java-only геттеры/сеттеры

Для очень горячих путей — Java-only аксессоры, не вызывающие Python:

```python
from extera_utils.classes import jfield, jgetmethod, jsetmethod

@java_subclass(SomeJavaClass)
class MySubclass(Base):
    shadow_value = jfield(
        "boolean", default=False,
        methods=[
            jgetmethod("isShadow"),
            jsetmethod("setShadow"),
        ],
    )
```

Генерирует поле `shadow_value`, метод `isShadow()` (возвращает поле) и `setShadow(boolean)` (пишет поле). Это же удобно для переопределения родительского геттера статичным значением: если у `SomeFactoryClass` уже есть `isShadow()`, сгенерированный Java-метод переопределит его и вернёт поле мгновенно.

## Конструкторы

Два этапа:

- `@jpreconstructor(...)` — **до** вызова родительского Java-конструктора;
- `@jconstructor(...)` — **после** создания Java-объекта.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @jpreconstructor(["java.lang.String"])
    def normalize_text(cls, text):
        return [text.strip()]  # None → оставить оригинал; список/tuple → замена; одиночное значение → для 1-арг конструктора

    initialized = jfield("boolean", default=False)

    @jconstructor(["java.lang.String"])
    def init_from_text(self, text):
        self.initialized = True
        self.last_text = text
```

Порядок инициализации:

1. `@jpreconstructor(...)` (если совпал)
2. Java-конструктор родителя
3. Python `__init__(...)`
4. совпавший `@jconstructor(...)`
5. `on_post_init(...)`

Если нужен один универсальный хук — `__init__` обычно достаточен:

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def __init__(self, text):
        self.python_only_state = text

    def on_post_init(self, text):
        print("Java object is ready:", self.java)
```

Для разных логик по перегрузкам конструктора — `@jconstructor(...)`.

## Доступ к Java-инстансу

Внутри управляемого подкласса:

- `self.java` — привязанный Java-инстанс;
- `self.this` — алиас `self.java`.

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    @joverride()
    def onAttached(self):
        print(self.java.getClass().getName())
```

Если Python не находит атрибут на peer-объекте, он фолбэчится на Java-инстанс: `self.getContext()`, `self.hashCode()` работают напрямую. Но если Java API проверяет точный runtime-тип — передавайте `instance.java`.

## Жизненный цикл объектов

Управляемый объект и его Java-инстанс живут, пока используется хотя бы одна сторона: зарегистрированный в Java объект (receiver, observer, view в layout) держит свой Python-peer и его состояние, а peer, на который ссылается ваш код, держит Java-инстанс. Когда обе стороны отпускают — всё освобождается. При выгрузке плагина объекты его классов освобождаются автоматически.

### Досрочное освобождение: `jrelease()`

Отсоединяет peer от Java-инстанса. Java-инстанс остаётся валидным, но его переопределённые методы ведут себя как у Java-базового класса. Полезно, когда Java может держать объект дольше, чем нужно вам:

```python
receiver = MyReceiver.new_instance()
context.registerReceiver(receiver.java, intent_filter)
...
receiver.jrelease()
```

Переопределите `on_release()` для очистки (Java-инстанс ещё подключён во время её выполнения):

```python
@java_subclass(SomeJavaClass)
class MySubclass(Base):
    def on_release(self):
        self.callback = None
```

> ⚠️ **Избегайте обратных ссылок на Java-инстанс.** Если peer хранит что-то, ссылающееся на его же Java-инстанс (`self.java` или Java-объект, держащий bound-метод peer), обе стороны держат друг друга до освобождения. Нужен Java-инстанс — берите из `self.java`, а не храните.

## Передача Python-объектов через Java

Низкоуровневый хелпер `PyObj` — когда нужно пронести произвольный Python-объект через Java-совместимые API:

```python
from extera_utils.classes import PyObj

payload = PyObj.create({"debug": True, "label": "factory args"})
```

Применение: непрозрачное плагинное состояние в Java-полях (например, `UItem.object`), низкоуровневые адаптеры и фабрики. Для обычного сабклассирования `PyObj` не нужен.

## Модификаторы Java-объектов (JA/JIR/JS…)

Удобное поведение встроено в exteraGram-форк Chaquopy — работает для любого Java-объекта или класса без импортов:

```python
obj.JA.privateField                    # доступ к приватному полю
obj.JIR.method1().method2()            # цепочка с игнором результатов
SomeJavaClass.JGS.someStaticProperty   # static геттер/сеттер
```

### Две группы модификаторов

**Постоянные** (меняют поведение самого объекта/класса):

- `JUseGetterAndSetter` / `JGS`, `JNotUseGetterAndSetter` / `JNGS`;
- `JAccessAll` / `JA`, `JNotAccessAll` / `JNA`.

**Скоуп-цепочки** (действуют только на текущую цепочку доступа):

- `JIgnoreResult` / `JIR`, `JNotIgnoreResult` / `JNIR`;
- `JSafe` / `JS`, `JNotSafe` / `JNS`.

### Поведение по умолчанию

- getter/setter-режим включён, access-all выключен, safe выключен, ignore-result выключен.

Это значит: обычный доступ пробует стандартное поведение Chaquopy, но «поле-подобные» имена предпочитают JavaBean-аксессоры:

```python
title = obj.title        # если есть getTitle() — используется он
obj.title = "Updated"    # если есть setTitle(...) — используется он
```

### Getter/Setter-режим

`JNGS` — сырой доступ к полям вместо `getX()`/`setX(...)`:

```python
obj.JNGS          # выключить для этого объекта
value = obj.result  # теперь читается само поле
obj.JGS           # включить обратно
```

Работает и на классах (для статических): `SomeJavaClass.JGS; caption = SomeJavaClass.caption`. Режим предназначен для поле-подобных имён (`title`, `result`, `messageObject`); имена, похожие на методы (`getClass()`, `isInterface()`), повторно не перенаправляются.

### AccessAll-режим

`JA` открывает непубличные declared-члены:

```python
obj.JA
secret = obj.secretField
obj.secretMethod()
obj.JNA  # выключить
```

И на классах: `SomeJavaClass.JA; token = SomeJavaClass.secretToken; SomeJavaClass.rebuildCache(); SomeJavaClass.JNA`. Внутри форк кеширует declared-методы/поля/вложенные классы на класс, поэтому `JA` не сканирует метаданные рефлексии каждый раз.

### IgnoreResult-режим

`JIR` — цепочка возвращает исходный receiver вместо Java-результата:

```python
obj.JIR.setTitle("Hello").requestLayout().invalidate()
value = obj.JIR.method1().method2().method3().JNIR.method4()  # method4 вернёт реальный результат
```

### Safe-режим

`JS` — опциональные безопасные цепочки: недостающий атрибут/метод в любом звене даёт `JNone` вместо исключения. Режим распространяется на возвращённые Java-объекты и превращает `None` в `JNone`:

```python
value = obj.JS.info.details.title        # None в середине — безопасно
value = obj.JS.missing.anything.call()   # JNone вместо исключения
```

### `JNone`

Сентинел safe-цепочек: falsy, продолжает проглатывать обращения к атрибутам и вызовы, сравнивается по идентичности:

```python
from java import JNone

result = obj.JS.maybeNull.value
if result is JNone:
    print("No value")
```

### Ограничения

Это удобный доступ, а не замена точной рефлексии: для контроля перегрузок используйте обычную рефлексию; непубличный поиск основан на declared-членах; getter/setter-предпочтение — только для поле-подобных имён.

## Полный пример

```python
from base_plugin import BasePlugin
from extera_utils.classes import (
    Base, java_subclass, jfield, joverride, joverload,
    jmethod, jconstructor, jpreconstructor,
)
from java.util import ArrayList


@java_subclass(ArrayList)
class DebugList(Base):
    add_calls = jfield("int", default=0)
    remove_calls = jfield("int", default=0)
    last_action = jfield("java.lang.String", default="created")

    @jpreconstructor(["int"])
    def normalize_capacity(cls, capacity):
        return [max(0, capacity)]

    def __init__(self, *args):
        self.history = []  # Python-only состояние, не Java-поле

    @jconstructor(["int"])
    def init_with_capacity(self, capacity):
        self.last_action = f"capacity:{capacity}"
        self.history.append(f"ctor(capacity={capacity})")

    @joverride()
    def size(self):
        current = super().size()
        self.history.append(f"size() -> {current}")
        return current

    @joverload("add", ["java.lang.Object"])
    def add_item(self, value):
        self.add_calls += 1
        self.last_action = f"add:{value}"
        return super().add_item(value)

    @joverload("add", ["int", "java.lang.Object"])
    def add_at_index(self, index, value):
        self.add_calls += 1
        return super().add_at_index(index, value)

    @joverload("remove", ["int"])
    def remove_at_index(self, index):
        self.remove_calls += 1
        return super().remove_at_index(index)

    @joverload("remove", ["java.lang.Object"])
    def remove_item(self, value):
        self.remove_calls += 1
        return super().remove_item(value)

    @jmethod("debugSummary", "java.lang.String", [])
    def debug_summary(self):
        return (
            f"size={self.size()}, add_calls={self.add_calls}, "
            f"remove_calls={self.remove_calls}, last_action={self.last_action}"
        )


class ProxyDebugPlugin(BasePlugin):
    def on_plugin_load(self):
        debug_list = DebugList.new_instance(4)  # выберет конструктор ArrayList(int)

        debug_list.add("one")
        debug_list.add("two")
        debug_list.add(1, "inserted")
        debug_list.remove("two")

        self.logger.info(f"Class name: {debug_list.getClass().getName()}")
        self.logger.info(f"Java size(): {debug_list.size()}")
        self.logger.info(f"add_calls: {debug_list.add_calls}")
        self.logger.info(f"debugSummary(): {debug_list.debugSummary()}")
        self.logger.info(f"Same object hash: {debug_list.java.hashCode()}")
```

## Пример: кастомная настройка через class proxy

SDK настроек сам использует эту систему для фабрик. Упрощённый паттерн с ячейкой списка плагинов:

```python
from client_utils import get_media_data_controller
from base_plugin import BasePlugin
from ui.settings import Text, Custom
from extera_utils.classes import Base, java_subclass, joverride

from android.util import TypedValue
from android.view import View, Gravity
from android.widget import LinearLayout, FrameLayout, TextView
from android.content import Context

from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.ActionBar import Theme
from org.telegram.ui.Components import LayoutHelper, BackupImageView, UItem

from com.exteragram.messenger.plugins import PluginsController, Plugin
from com.exteragram.messenger.plugins.models import CustomSetting

MATCH_PARENT = -1
WRAP_CONTENT = -2


@java_subclass(FrameLayout)
class PluginCell(Base):
    def on_post_init(self, context: Context):
        self.need_divider = False
        self.setWillNotDraw(False)

        self.icon_view = BackupImageView(context)
        self.icon_view.setRoundRadius(AndroidUtilities.dp(8))
        self.addView(self.icon_view, LayoutHelper.createFrame(48, 48, Gravity.LEFT | Gravity.CENTER_VERTICAL, 16, 0, 0, 0))

        text_layout = LinearLayout(context)
        text_layout.setOrientation(LinearLayout.VERTICAL)

        self.name_text_view = TextView(context)
        self.name_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteBlackText))
        self.name_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
        self.name_text_view.setSingleLine(True)

        self.author_text_view = TextView(context)
        self.author_text_view.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteGrayText))
        self.author_text_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        self.author_text_view.setSingleLine(True)

        text_layout.addView(self.name_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT))
        text_layout.addView(self.author_text_view, LayoutHelper.createLinear(MATCH_PARENT, WRAP_CONTENT, 0, 2, 0, 0))
        self.addView(text_layout, LayoutHelper.createFrame(MATCH_PARENT, WRAP_CONTENT, Gravity.LEFT | Gravity.CENTER_VERTICAL, 76, 0, 16, 0))

    def set_data(self, plugin: Plugin, divider: bool):
        self.name_text_view.setText(plugin.getName() or "Unknown Plugin")
        version = plugin.getVersion() or "1.0"
        author = plugin.getAuthor() or "Unknown"
        self.author_text_view.setText(f"v{version} • {author}")

        show_icon = plugin.getPack() is not None and plugin.getIndex() >= 0
        pack, index = (plugin.getPack(), plugin.getIndex()) if show_icon else ("exteraPlugins", 0)
        get_media_data_controller().setPlaceholderImageByIndex(self.icon_view, pack, index, "100_100")
        self.requestLayout()

    @joverride()
    def onMeasure(self, widthMeasureSpec: int, heightMeasureSpec: int):
        super().onMeasure(
            View.MeasureSpec.makeMeasureSpec(View.MeasureSpec.getSize(widthMeasureSpec), View.MeasureSpec.EXACTLY),
            View.MeasureSpec.makeMeasureSpec(AndroidUtilities.dp(68), View.MeasureSpec.EXACTLY),
        )


@java_subclass(CustomSetting.Factory)
class PluginCellSettingFactory(Base):
    @joverride()
    def create(self, plugin: Plugin, setting: CustomSetting, args):
        if not args:
            return None
        item = UItem.ofFactory(self.java_class())
        item.object = args
        return item

    @joverride()
    def createView(self, context, list_view, current_account, class_guid, resources_provider):
        view = PluginCell.new_instance(context)
        return view.java

    @joverride()
    def bindView(self, view, item, divider, adapter, list_view):
        if not item.object:
            return False
        PluginCell.from_java(view).set_data(item.object, divider)


class TestPlugin(BasePlugin):
    def on_plugin_load(self):
        self.pcsfi = PluginCellSettingFactory.new_java_instance()
        UItem.UItemFactory.setup(self.pcsfi)

    def create_settings(self):
        plugins = PluginsController.getInstance().plugins.values().toArray()
        if not plugins:
            return [Text("Restart plugin please, no plugins found")]

        return [
            Custom(factory=self.pcsfi, factory_args=_p) for _p in plugins
        ]
```

`new_instance()` даёт Python-peer; в API, требующие `CustomSetting.Factory`, передавайте `factory.java`.

## Продвинутые заметки

- **MRO:** для управляемых override прокси-система ставит Python-бридж, чтобы `super()` звал родительскую Java-реализацию. Используйте `super()` только для методов с `@joverride(...)` / `@joverload(...)`.
- **Диспетчеризация сигнатур:** Java→Python вызовы ключуются полной сигнатурой вида `methodName:int:java.lang.String` — это и делает перегрузки рабочими.
- **Примитивные типы:** `void`, `boolean`, `byte`, `char`, `short`, `int`, `long`, `float`, `double`.

### Когда использовать class proxy

✅ Используйте, когда: Java API требует именно подкласс (а не интерфейсный прокси); нужны вызовы родителя через `super()`; нужно поведение по перегрузкам; нужны Java-поля или дополнительные Java-методы.

❌ Не используйте, когда достаточно простого `dynamic_proxy(...)` для интерфейса. И убедитесь, что базовый Java-класс вообще сабклассируется — **final-классы нельзя** использовать как базу прокси.

## Troubleshooting

| Проблема | Проверить |
|---|---|
| `super()` не работает | Метод объявлен через `@joverride(...)`/`@joverload(...)`; имя/сигнатура совпадают с Java-родителем |
| Оверлоад не вызывается | Верное Java-имя; типы аргументов совпадают с реальной сигнатурой; примитивы — примитивными именами (`"int"`), не обёртками |
| Хук конструктора не вызывается | Сигнатура в `@jpreconstructor(...)`/`@jconstructor(...)` совпадает с реальным конструктором |
| Значение поля всегда `None` | Поле объявлено `jfield(...)`; тип валиден; default совместим с Java-типом |
| Генерация класса падает сразу | Базовый класс не `final`; сигнатуры конструктора существуют; сигнатуры оверлоадов совпадают с типами аргументов Java-методов |
