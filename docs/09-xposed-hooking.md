# 09. Xposed-хуки Java-методов

Перехват Java-методов и конструкторов из Python: `MethodHook`, `MethodReplacement`, `BaseHook` и фильтры.

## Введение

Xposed-хуки позволяют перехватывать вызовы методов внутри приложения: менять аргументы, инспектировать или подменять возвращаемые значения, пропускать оригинальную реализацию. Одна из самых мощных частей SDK — и одна из самых простых для неправильного использования.

## Три стиля хук-хендлеров

### 1. `MethodHook` — до/после

Когда нужны `before_hooked_method(...)` и/или `after_hooked_method(...)`:

```python
from base_plugin import MethodHook


class TitleLoggerHook(MethodHook):
    def before_hooked_method(self, param):
        title = param.args[0]
        print(f"ActionBar title is about to become: {title}")

    def after_hooked_method(self, param):
        print(f"Method finished with result: {param.getResult()}")
```

### 2. `MethodReplacement` — полная замена

```python
from base_plugin import MethodReplacement


class TitleReplacer(MethodReplacement):
    def replace_hooked_method(self, param):
        print("Original method was replaced.")
        return None
```

### 3. Функциональные хуки / `BaseHook`

Для простых случаев не нужен класс — передайте `before=` и/или `after=` напрямую:

```python
self.hook_method(
    some_method,
    before=lambda param: self.logger.info("Method is about to run!"),
    after=lambda param: self.logger.info(f"Finished with result: {param.getResult()}"),
)
```

Внутри SDK оборачивает этот стиль в `BaseHook` — небольшой подкласс `MethodHook`.

## Объект `param`

Все колбэки получают `de.robv.android.xposed.XC_MethodHook.MethodHookParam`:

| Член | Описание |
|---|---|
| `param.thisObject` | Инстанс, на котором вызван метод; `None` для статических |
| `param.args` | Массив аргументов — можно читать и **изменять** |
| `param.method` | Перехваченный `Method` или `Constructor` |
| `param.getResult()` | Возвращённое значение (доступно после выполнения) |
| `param.setResult(value)` | Установить результат вручную |

Если вызвать `param.setResult(...)` внутри `before_hooked_method(...)` — **оригинальный метод будет пропущен**:

```python
def before_hooked_method(self, param):
    if param.args[0] == "blocked":
        param.setResult(None)
```

## Фильтры

Фильтры решают, должен ли колбэк выполниться вообще. Два стиля:

- `@hook_filters(...)` на методах классовых хуков;
- `before_filters=[...]` и `after_filters=[...]` для функциональных.

```python
from base_plugin import HookFilter

self.hook_method(
    some_method,
    before=lambda p: self.logger.info("Arg 0 is null!"),
    before_filters=[HookFilter.ArgumentIsNull(0)],
)
```

### Доступные `HookFilter`

- `RESULT_IS_NULL`, `RESULT_IS_TRUE`, `RESULT_IS_FALSE`, `RESULT_NOT_NULL`
- `ResultIsInstanceOf(clazz)`
- `ResultEqual(value)`, `ResultNotEqual(value)`
- `ArgumentIsNull(index)`, `ArgumentNotNull(index)`
- `ArgumentIsFalse(index)`, `ArgumentIsTrue(index)`
- `ArgumentIsInstanceOf(index, clazz)`
- `ArgumentEqual(index, value)`, `ArgumentNotEqual(index, value)`
- `Condition(condition, object=None)` — MVEL-условие
- `Or(*filters)`

### Классовый пример

```python
from base_plugin import HookFilter, MethodHook, hook_filters


class ExampleHook(MethodHook):
    @hook_filters(HookFilter.ArgumentIsNull(0))   # только если первый аргумент null
    def before_hooked_method(self, param):
        ...

    @hook_filters(HookFilter.RESULT_NOT_NULL)     # только если результат не null
    def after_hooked_method(self, param):
        ...
```

### MVEL-условие

```python
from base_plugin import HookFilter, MethodHook, hook_filters


class ConditionalHook(MethodHook):
    @hook_filters(
        HookFilter.Condition(
            "param.args[0] == object || this instanceof android.view.View",
            object=500,
        )
    )
    def before_hooked_method(self, param):
        ...
```

## Процесс хукинга

### Шаг 1. Найдите целевой метод или конструктор

Обычно начинают с `find_class(...)`, затем обычная Java-рефлексия прямо на возвращённом `Class`:

```python
from hook_utils import find_class

ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if not ActionBarClass:
    self.logger.info("ActionBar class not found!")
    return

# Метод: public void setTitle(CharSequence title)
CharSequenceClass = find_class("java.lang.CharSequence")
set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)

# Конструктор: public ActionBar(Context context)
ContextClass = find_class("android.content.Context")
action_bar_constructor = ActionBarClass.getDeclaredConstructor(ContextClass)
action_bar_constructor.setAccessible(True)
```

> ⚠️ **Не вызывайте `getClass()` на объекте `Class`.** `find_class(...)` уже возвращает Java `Class` — вызывайте `getDeclaredMethod(...)` / `getDeclaredConstructor(...)` прямо на нём.

### Шаг 2. Реализуйте хендлер

```python
from base_plugin import MethodHook, MethodReplacement


class TitleLoggerHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        title = param.args[0]
        self.plugin.log(f"ActionBar title is being set to: {title}")
        param.args[0] = f"[Hooked] {title}"

    def after_hooked_method(self, param):
        self.plugin.log("ActionBar title has been set.")


class TitleReplacer(MethodReplacement):
    def __init__(self, plugin):
        self.plugin = plugin

    def replace_hooked_method(self, param):
        self.plugin.log("ActionBar.setTitle() was replaced.")
        return None
```

### Шаг 3. Примените хук

```python
ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
CharSequenceClass = find_class("java.lang.CharSequence")

set_title_method = ActionBarClass.getDeclaredMethod("setTitle", CharSequenceClass)
set_title_method.setAccessible(True)

handler_instance = TitleLoggerHook(self)
self.unhook_obj = self.hook_method(set_title_method, handler_instance, priority=10)

if self.unhook_obj:
    self.logger.info("Successfully hooked ActionBar.setTitle()")
```

Хуки снимаются автоматически при выгрузке плагина, но можно и вручную.

### Шаг 4. Все методы или все конструкторы

```python
MyViewClass = find_class("com.example.MyCustomView")
unhook_list = self.hook_all_methods(MyViewClass, "onMeasure", TitleLoggerHook(self))
if unhook_list:
    self.logger.info(f"Hooked {len(unhook_list)} onMeasure overload(s).")

SomeClass = find_class("com.example.SomeClass")
constructor_unhooks = self.hook_all_constructors(SomeClass, TitleLoggerHook(self))
```

### Шаг 5. Ручное снятие

```python
if self.unhook_obj:
    self.unhook_method(self.unhook_obj)
    self.unhook_obj = None
```

После `hook_all_methods(...)` / `hook_all_constructors(...)` пройдитесь по возвращённому списку и снимите каждый элемент.

## Практические примеры

### Пример 1: изменить аргументы до вызова

```python
from base_plugin import MethodHook
from hook_utils import find_class
from java import jint


class ToastHook(MethodHook):
    def before_hooked_method(self, param):
        # Сигнатура: makeText(Context context, CharSequence text, int duration)
        original_text = param.args[1]
        param.args[1] = f"(Plugin) {original_text}"


ToastClass = find_class("android.widget.Toast")
ContextClass = find_class("android.content.Context")
CharSequenceClass = find_class("java.lang.CharSequence")

make_text_method = ToastClass.getDeclaredMethod("makeText", ContextClass, CharSequenceClass, jint)
self.hook_method(make_text_method, ToastHook())
```

### Пример 2: изменить возвращаемое значение

```python
class BuildVarsHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        original_result = param.getResult()
        self.plugin.log(f"Original result: {original_result}")
        param.setResult(False)


BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
is_main_app_method = BuildVarsClass.getDeclaredMethod("isMainApp")
self.hook_method(is_main_app_method, BuildVarsHook(self))
```

### Пример 3: пропустить оригинальный вызов

```python
from java.lang import Boolean, Long


class FormatFileSizeHook(MethodHook):
    def before_hooked_method(self, param):
        size = param.args[0]
        if size < 1024:
            param.setResult(f"{size} bytes (edited)")  # оригинал не выполнится


AndroidUtilitiesClass = find_class("org.telegram.messenger.AndroidUtilities")
format_file_size_method = AndroidUtilitiesClass.getDeclaredMethod(
    "formatFileSize", Long.TYPE, Boolean.TYPE, Boolean.TYPE,
)
self.hook_method(format_file_size_method, FormatFileSizeHook())
```

### Пример 4: полностью заменить метод

```python
from java.lang import String as JString

from base_plugin import MethodReplacement
from hook_utils import find_class


class NoOpLogger(MethodReplacement):
    def replace_hooked_method(self, param):
        return None  # оригинальный метод никогда не вызывается


FileLogClass = find_class("org.telegram.messenger.FileLog")
log_method = FileLogClass.getDeclaredMethod("d", JString)
self.hook_method(log_method, NoOpLogger())
```

> 💡 **Возвращаемые значения в `MethodReplacement`.** Ваш `replace_hooked_method(...)` — вся реализация. Верните значение, совместимое с оригинальным Java-методом:
> - `None` — для `void`;
> - Python `bool` / `int` / `float` — для примитивных типов;
> - совместимый Java- или Python-объект — для ссылочных типов.

## Куда дальше

Для генерации настоящих Java-подклассов (с полями, конструкторами, `super()`, оверлоадами) нужен **Class Proxy** — см. [10. Class Proxy](10-class-proxy.md).
