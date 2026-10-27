# 08. Файлы, Intents и рефлексия

`file_utils`, `FilesController`, `IntentsManager` и `hook_utils`.

## Часть 1. `file_utils`

Две зоны ответственности:

- файловые хелперы для типовых задач плагина;
- `FilesController` — перехват открытия файлов по расширению + кастомные иконки.

### Стандартные директории

```python
from file_utils import (
    get_plugins_dir,   # папка файлов плагинов
    get_cache_dir,     # кеш Telegram
    get_files_dir,
    get_images_dir,
    get_videos_dir,
    get_audios_dir,
    get_documents_dir,
)

plugins_path = get_plugins_dir()
```

### Операции с директориями

**`ensure_dir_exists(path)`** — создаёт директорию (включая родителей), если её нет:

```python
import os
from file_utils import ensure_dir_exists, get_plugins_dir

data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(data_dir)
```

**`list_dir(...)`** — список файлов/директорий с рекурсией и фильтром по расширениям:

```python
from file_utils import get_cache_dir, get_images_dir, list_dir

image_files = list_dir(path=get_images_dir(), extensions=[".jpg", ".png"])

cache_dirs = list_dir(path=get_cache_dir(), recursive=True, include_files=False, include_dirs=True)
```

- `recursive=False` — обходить вложенные папки;
- `include_files=True` / `include_dirs=False` — что включать;
- `extensions=None` — суффикс-фильтр вида `[".json", ".txt"]`.

### Операции с файлами

| Функция | Что делает | При ошибке |
|---|---|---|
| `write_file(path, content: str)` | Пишет текст (перезаписывает) | — |
| `read_file(path)` | Читает весь файл как текст | `None` + лог |
| `write_file_bytes(path, content: bytes)` | Пишет бинарные данные | — |
| `read_file_bytes(path)` | Читает весь файл как bytes | `None` + лог |
| `delete_file(path)` | Удаляет файл | `False` |

```python
import os
from file_utils import ensure_dir_exists, get_plugins_dir, write_file, read_file, delete_file

data_dir = os.path.join(get_plugins_dir(), "my_plugin_data")
ensure_dir_exists(data_dir)

write_file(os.path.join(data_dir, "config.txt"), "enabled=true")
config_text = read_file(os.path.join(data_dir, "config.txt"))

was_deleted = delete_file("/path/to/temp.txt")
```

### `FilesController` — перехват открытия файлов

Регистрирует обработчики расширений: когда пользователь открывает файл зарегистрированного типа, ваш колбэк выполняется вместо стандартного сценария. Применение: кастомные вьюверы, плагинные архивы, свои иконки расширений.

> 🔑 `FilesController.register(...)` возвращает **секретную строку** — она нужна для `unregister(...)`.

#### `FilesController.SUPPORT_ICONS`

`True`, если текущая версия клиента поддерживает кастомные иконки файлов.

#### `FilesController.Place` — откуда пришло открытие

`UNKNOWN`, `ChatActivity`, `FilteredSearchView`, `SharedMediaLayout`, `SearchDownloadsContainer`, `ChannelAdminLogActivity`.

#### `FilesController.FileInfo`

| Поле | Описание |
|---|---|
| `ext: str` | Расширение |
| `on_click: Callable[[OnClickArgs], None]` | Обработчик |
| `whitelist_places: list[Place] = []` | Только в этих местах |
| `blacklist_places: list[Place] = []` | Везде, кроме этих мест |
| `get_icon: Optional[Callable[[], Drawable]]` | Фабрика иконки (нужен `SUPPORT_ICONS`) |

Нельзя использовать `whitelist_places` и `blacklist_places` одновременно.

#### `OnClickArgs`

Поля: `place`, `file`, `file_name`, `message`, `activity`, `parent_fragment`.

#### Регистрация

```python
from file_utils import FilesController


def on_zip_click(args: FilesController.OnClickArgs):
    print("Opened:", args.file_name)
    print("From:", args.place)
    print("Absolute path:", args.file.getAbsolutePath())


secret = FilesController.register(
    FilesController.FileInfo(ext="zip", on_click=on_zip_click)
)
```

#### Ограничение по месту

```python
# Только внутри чатов:
secret = FilesController.register(
    FilesController.FileInfo(
        ext="txt", on_click=on_txt_click,
        whitelist_places=[FilesController.Place.ChatActivity],
    )
)

# Везде, кроме контейнера загрузок:
secret = FilesController.register(
    FilesController.FileInfo(
        ext="pdf", on_click=on_pdf_click,
        blacklist_places=[FilesController.Place.SearchDownloadsContainer],
    )
)
```

#### Иконка расширения

```python
if FilesController.SUPPORT_ICONS:
    secret = FilesController.register(
        FilesController.FileInfo(ext="zip", on_click=on_zip_click, get_icon=make_zip_icon)
    )
```

#### Отмена регистрации

```python
FilesController.unregister("zip", secret)
```

#### Исключения

- `FilesController.ExtensionAlreadyRegistered` — расширение уже занято;
- `FilesController.ExtensionNotRegistered`;
- `FilesController.SecretInvalid` — неверный secret.

## Часть 2. Intents (`intents`)

`IntentsManager` — глобальный API для реакции на ссылки и intents, обрабатываемые exteraGram. **Доступен только на exteraGram 66999 (12.6.4) и новее.**

```python
from intents import IntentsManager as IM
```

### `new_global_before_handler(...)` / `new_global_after_handler(...)`

```python
def on_chat_link(id):
    print("Chat id:", id)

handle = IM.new_global_before_handler(
    on_chat_link,
    scheme="tg",
    host="chat",
    required_path_args_names=["id"],
)

def on_after_link(intent=None, scheme=None, host=None):
    print("Handled:", scheme, host)

handle = IM.new_global_after_handler(
    on_after_link,
    scheme="https",
    host="t.me",
)
```

### Фильтры (одинаковы для before/after)

| Параметр | Сопоставление |
|---|---|
| `scheme` | `intent.data.scheme` |
| `host` | `intent.data.host` |
| `path` | `intent.data.path`; если не начинается с `/` — добавится автоматически. Поддерживает шаблоны `{user_id}` |
| `required_path_args_names` | ⚠️ Название историческое: это **обязательные query-аргументы**. Если хотя бы одного нет — хендлер не запустится |
| `action` | `intent.getAction()` |
| `whitelist_flags` | Все перечисленные флаги должны быть в `intent.getFlags()` |
| `blacklist_flags` | Ни один из флагов не должен присутствовать |
| `type` | `intent.getType()` (например `"text/plain"`) |
| `categories` | Все категории должны присутствовать в `intent.getCategories()` |
| `priority` | Порядок выполнения по возрастанию; меньше = раньше |

```python
# Шаблон пути с переменной:
def on_profile(user_id, path=None):
    print("User:", user_id, "Path:", path)

IM.new_global_before_handler(on_profile, scheme="tg", host="profile", path="/user/{user_id}")

# Action:
from android.content import Intent
IM.new_global_before_handler(lambda **kw: None, action=Intent.ACTION_VIEW)
```

### Аргументы колбэка

SDK инспектирует сигнатуру и передаёт **только то, что функция явно запрашивает**.

Стандартные имена: `intent`, `scheme`, `host`, `path`, `query_args`, `action`, `flags`, `type`, `categories`. Дополнительно:

- каждый query-аргумент по имени (`id`, `plugin`, …);
- каждая переменная path-шаблона (`user_id` из `/user/{user_id}`).

```python
def on_setting_link(plugin, section=None):
    print("Plugin:", plugin, "Section:", section)

IM.new_global_after_handler(on_setting_link, scheme="https")

# **kwargs — весь именованный контекст:
def on_any_link(**kwargs):
    print(kwargs.get("scheme"), kwargs.get("id"), kwargs.get("query_args"))

IM.new_global_before_handler(on_any_link, scheme="tg")

# *args — стандартный позиционный набор из 9 значений:
def on_any_link(*args):
    intent, scheme, host, path, query_args, action, flags, type, categories = args

IM.new_global_before_handler(on_any_link, scheme="tg")
```

### Прерывание обработки

**Только `before`-хендлеры** могут остановить и остальные Python-хендлеры, и оригинальную Java-обработку intent: верните `True`. SDK выполнит `param.setResult(true)` в Xposed-хуке. Любое другое значение обработку не прерывает. `after`-хендлеры прерывать не могут.

```python
def intercept_chat(id):
    print("Intercepted:", id)
    return True

IM.new_global_before_handler(intercept_chat, scheme="tg", host="chat", required_path_args_names=["id"])
```

### `HandlerHandle` и снятие

Оба метода регистрации возвращают `HandlerHandle`:

```python
handle = IM.new_global_before_handler(on_link, scheme="tg")
handle.unhandle()
# или по id:
IM.unhandle(handle.handler_id)  # исключение IM.HandlerNotRegistered, если id не существует
```

### `IM.parse(url)`

Разбор URL-подобных строк:

```python
parts = IM.parse("tg://chat?id=123")
parts["scheme"]  # "tg"
parts["host"]    # "chat"
parts["query"]   # "id=123"
# также может включать: path, query
```

### Заметки

- сопоставление по `scheme`/`host`/`path`/query-аргументам/`action`/флагам/`type`/`categories` происходит **на Java-стороне** — в Python диспетчеризуются только совпавшие хендлеры;
- хендлеры выполняются в порядке возрастания `priority`;
- `required_path_args_names` означает именно обязательные query-аргументы.

## Часть 3. Рефлексия (`hook_utils`)

Инструменты для поиска Java-классов и доступа к приватным полям и методам через рефлексию. Особенно полезны для Xposed-хуков, когда нужен доступ к непубличным членам класса.

> ⚠️ **Осторожно.** Рефлексия мощная, но хрупкая: ломается при изменениях кода приложения. Всегда оборачивайте в `try/except` и проверяйте `None`.

### `find_class(class_name: str)`

Безопасно находит Java-класс по полному имени. Возвращает `Class`-объект или `None`:

```python
from hook_utils import find_class

ActionBarClass = find_class("org.telegram.ui.ActionBar.ActionBar")
if ActionBarClass:
    self.logger.info(f"Found: {ActionBarClass.getName()}")
```

### `get_private_field(obj, field_name)`

Значение приватного (или публичного) поля инстанса; ищет по всей иерархии классов. Возвращает значение или `None`:

```python
from hook_utils import get_private_field

chat_list_view = get_private_field(chatActivity, "chatListView")
```

### `set_private_field(obj, field_name, new_value)`

Меняет значение поля инстанса. Возвращает `True`/`False`:

```python
success = set_private_field(user_object, "verified", True)
```

### `get_static_private_field(clazz, field_name)`

Значение статического поля класса (`clazz` — результат `find_class` или `java.lang.Class`):

```python
ExteraConfigClass = find_class("com.exteragram.messenger.ExteraConfig")
if ExteraConfigClass:
    config_loaded = get_static_private_field(ExteraConfigClass, "configLoaded")
```

### `set_static_private_field(clazz, field_name, new_value)`

Меняет статическое поле. Возвращает `True`/`False`:

```python
BuildVarsClass = find_class("org.telegram.messenger.BuildVars")
if BuildVarsClass:
    set_static_private_field(BuildVarsClass, "DEBUG_VERSION", True)
```
