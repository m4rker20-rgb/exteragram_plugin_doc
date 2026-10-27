# 07. Логирование (`log_utils`)

Логгер с уровнями, тегами по плагину, трейсбеками, дампами объектов, таймерами и хендлерами.

`log_utils` — логгер SDK. Каждый плагин получает его как `self.logger`; записи идут в logcat под тегом `plugin.<id плагина>` — можно фильтровать вывод своего плагина без grep.

```python
from base_plugin import BasePlugin


class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info("Loaded version %s", self.version)
        self.logger.debug("Settings: %s", self.export_settings())
```

Просмотр:

```bash
adb logcat -s plugin.my_plugin
```

Каждая запись начинается с места логирования:

```
I plugin.my_plugin: [my_plugin.py:on_plugin_load:6] Loaded version 1.0
```

> ⚠️ `android_utils.log()` и `BasePlugin.log()` работают, но это legacy: всегда debug-уровень и неотфильтровываемый тег. См. «Миграция с log()» ниже.

## Уровни

| Уровень | Метод | Для чего |
|---|---|---|
| `LogLevel.VERBOSE` | `logger.verbose(...)` | Очень болтливый трейсинг (например, каждый вызов хука) |
| `LogLevel.DEBUG` | `logger.debug(...)` | Детали разработки |
| `LogLevel.INFO` | `logger.info(...)` | Нормальные события жизненного цикла |
| `LogLevel.WARNING` | `logger.warning(...)` | Неожиданности, которые плагин пережил |
| `LogLevel.ERROR` | `logger.error(...)`, `logger.exception(...)` | Сбои |

Уровни мапятся на приоритеты Android, поэтому `adb logcat *:E` покажет только ошибки.

По умолчанию логгер показывает **DEBUG и выше** при включённом developer mode и **INFO и выше** в противном случае — пользователи релизной версии не платят за debug-логи.

Изменение уровня:

```python
from log_utils import LogLevel

self.logger.level = LogLevel.VERBOSE  # принимает и "verbose", и число
self.logger.level = LogLevel.OFF      # заглушить
self.logger.level = None              # вернуть дефолт
```

> 💡 **Передавайте аргументы вместо f-строк.** `self.logger.debug("chat %s: %s", chat_id, message)` форматирует сообщение только при включённом DEBUG. f-строка строится всегда — это важно в хуках, которые вызываются тысячи раз.

## Ошибки

`exception()` внутри `except` добавит трейсбек автоматически:

```python
try:
    self.process(update)
except Exception:
    self.logger.exception("Failed to process %s", update_name)
```

Вне `except` передайте исключение явно:

```python
self.logger.error("Request failed", exc_info=error)
```

## Инспекция объектов: `dump()`

`dump()` печатает структуру объекта: Telegram `TLRPC`-объекты, другие Java-объекты и Python-значения.

```python
self.logger.dump(user, "user")              # на DEBUG
self.logger.dump(message.messageOwner, depth=3)
```

Пример вывода:

```
user: TLRPC.TL_user @3f2a1b
  ...
  first_name = 'Ivan'
  id = 123456789
  ...
  photo = TLRPC.TL_userProfilePhoto @9c01de
    dc_id = 2
    photo_id = 5221840139847234
    stripped_thumb = byte[412]
```

Как показываются значения:

- читаются **только поля** — геттеры, свойства и `__getattr__` не вызываются, дамп без побочных эффектов;
- Java-объекты обходятся по иерархии классов (без статических полей); классы из `java.*`, `android.*` и платформенных пакетов показываются через `toString()`;
- списки/карты/массивы/sparse-массивы показывают размер и первые `max_items` элементов (по умолчанию 20);
- вложенность обрезается на `depth` уровнях (по умолчанию 2) с пометкой `{…}`; самоссылки помечаются `<cycle>`.

`dump()` — также обычная функция, возвращающая строку:

```python
from android_utils import copy_to_clipboard
from log_utils import dump

copy_to_clipboard(dump(message.messageOwner, depth=4))
```

## Дочерние логгеры

Разбейте большой плагин на зоны со своими уровнями:

```python
self.net_log = self.logger.child("net")    # тег: plugin.my_plugin.net
self.net_log.level = LogLevel.VERBOSE
```

Дочерний логгер наследует уровень родителя, пока не задан свой.

В хелпер-модулях получайте логгер без прокидывания плагина:

```python
from log_utils import get_logger

logger = get_logger(__name__)
```

`get_logger()` определяет, какой плагин вызвал, по файлу вызова. В главном файле плагина вернёт тот же логгер, что и `self.logger`; в Elyx-модулях — дочерний с именем модуля.

## Замер времени

```python
with self.logger.timed("Load stickers"):
    self.load_stickers()
# D plugin.my_plugin: [my_plugin.py:on_plugin_load:21] Load stickers took 12.4 ms
```

Если блок кидает исключение — запись завершится `(failed)`, а исключение продолжит распространяться.

## Хендлеры

Хендлер получает каждую запись логгера и его детей после фильтрации по уровню:

```python
from android_utils import run_on_ui_thread
from log_utils import LogLevel, LogRecord
from ui.bulletin import BulletinHelper


class MyPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.add_handler(self.show_errors)

    def show_errors(self, record: LogRecord):
        if record.level >= LogLevel.ERROR:
            run_on_ui_thread(lambda: BulletinHelper.show_error(record.message))
```

Поля `LogRecord`: `level`, `tag`, `message`, `location` (`file:function:line` или `None`), `traceback` (`str` или `None`), `created` (Unix-время), `text` (запись ровно как в logcat).

Правила:

- хендлеры выполняются синхронно в потоке логирования — держите их быстрыми;
- логирование изнутри хендлера не вызывает хендлеры повторно;
- исключение в хендлере логируется под тегом `pysdk` и не влияет на плагин;
- при выгрузке плагина SDK снимает его хендлеры и сбрасывает уровни.

## `stacklevel` для хелперов

Если вы оборачиваете логгер своим хелпером, передайте `stacklevel=2`, чтобы локация указывала на вызывающего, а не на хелпер:

```python
def warn_once(self, key, text):
    if key not in self.warned:
        self.warned.add(key)
        self.logger.warning(text, stacklevel=2)
```

## Длинные сообщения

Logcat режет запись на ~4 КБ. Логгер сам разбивает длинные сообщения (большие дампы, трейсбеки) на несколько записей по границам строк.

## Миграция с `log()`

| Было | Стало |
|---|---|
| `self.log("Loaded")` | `self.logger.info("Loaded")` |
| `log(f"Value: {x}")` | `logger.debug("Value: %s", x)` |
| `log(user)` | `logger.dump(user)` |
| `log(f"Error: {e}\n{traceback.format_exc()}")` | `logger.exception("Error")` |

Старые функции работают. `log(obj)` с нетекстовым значением теперь печатает то же, что `dump()`. Если плагин сам назначал `self.logger` — его значение вернётся вместо логгера SDK.

## Справочник

```python
class Logger:
    tag: str
    level: LogLevel | None            # None: уровень родителя или дефолт
    effective_level: LogLevel

    def verbose(msg, *args, exc_info=None, stacklevel=1)
    def debug(msg, *args, exc_info=None, stacklevel=1)
    def info(msg, *args, exc_info=None, stacklevel=1)
    def warning(msg, *args, exc_info=None, stacklevel=1)
    def error(msg, *args, exc_info=None, stacklevel=1)
    def exception(msg, *args, stacklevel=1)
    def log(level, msg, *args, exc_info=None, stacklevel=1)
    def dump(obj, label=None, *, level=LogLevel.DEBUG, depth=2, max_items=20, stacklevel=1)
    def timed(label, level=LogLevel.DEBUG)   # context manager
    def is_enabled_for(level) -> bool
    def child(name) -> Logger
    def add_handler(handler)
    def remove_handler(handler) -> bool

def get_logger(name: str | None = None) -> Logger
def dump(obj, depth=2, max_items=20) -> str
```

`Logger(tag)` создаёт автономный логгер с любым тегом, если он нужен вне иерархии плагина.
