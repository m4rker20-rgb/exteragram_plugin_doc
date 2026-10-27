# 11. Elyx: структурированные плагины

Многофайловые плагины с ресурсами, локализацией, зависимостями и live reload.

## Что такое Elyx

**Elyx** — структурированный формат плагинов exteraGram. Сохраняет знакомый `BasePlugin` API, но снимает требование уместить весь проект в один Python-файл.

Elyx-плагин может содержать:

- любое число Python-модулей и пакетов;
- изображения, SVG, Lottie-анимации, JSON, YAML и другие ресурсы;
- отдельные файлы локализации;
- бандл-колёса (`.whl`);
- метаданные и конфигурацию путей вне исходного кода;
- рабочий процесс десктоп-разработки с перезагрузкой на подключённом телефоне.

Установленный плагин — обычный подкласс `BasePlugin`. Elyx отвечает за поиск его файлов, создание плагин-специфичного окружения, загрузку зависимостей и изоляцию локальных импортов.

### Когда использовать Elyx

**Обычный однофайловый плагин** — если реализация маленькая и без ресурсов. **Elyx** — если хотя бы одно из:

- код естественно разросся в несколько модулей;
- UI нужны бандл-изображения, анимации, конфиги;
- тексты нужно переводить без правки Python-кода;
- нужны локальные библиотеки или повторяемая сборка;
- над плагином работает несколько человек;
- удобнее быстрая синхронизация файлов, чем переустановка `.py`.

### Ментальная модель

Elyx-архив — ZIP-совместимый файл с расширением `.elyx` или `.eaf`. В его корне лежит `refmap`-файл, указывающий рантайму на остальной проект:

```
my_plugin.elyx
├── refmap.yml
├── main.py
├── metainfo.yml
├── src/
│   ├── __init__.py
│   └── feature.py
├── assets/
│   └── logo.svg
└── strings/
    ├── strings_en.yml
    └── strings_ru.yml
```

В рантайме плагин импортирует окружение из одного стабильного публичного модуля:

```python
from elyx import assets, metainfo, refmap, settings, strings
```

> ⚠️ Импортируйте **только** `elyx`. Модули реализации рантайма — приватные и могут меняться без гарантий совместимости.

Elyx не заменяет остальной SDK: жизненный цикл, хуки, пункты меню, запросы и настройки по-прежнему из обычного SDK (`BasePlugin`, `ui.settings`, `client_utils`, class proxy, Xposed-хуки). Elyx добавляет структуру проекта и стабильное окружение вокруг этих API.

## Структура проекта и refmap

Elyx сознательно разделяет физический расклад и рантайм-значения. Можно держать маленький плагин в корне:

```
my_plugin/
├── refmap.yml
├── metainfo.yml
├── main.py
├── assets/
└── strings/
```

или разложить под вложенной директорией:

```
my_plugin/
├── refmap.yml
├── plugin/
│   ├── meta.yml
│   ├── src/
│   │   ├── __init__.py
│   │   └── main.py
│   ├── locales/
│   └── res/
└── wheels/
```

### Поддерживаемые форматы refmap

В корне архива Elyx находит первый из: `refmap.yaml`, `refmap.yml`, `refmap.json`. Рекомендуется YAML — лаконичен и не содержит исполняемого кода:

```yaml
metainfo: plugin/meta.yml
main: plugin/src/main.py
strings: plugin/locales
assets: plugin/res
wheels: wheels
```

### Ключи рантайма

| Ключ | Значение |
|---|---|
| `main` | Путь к входному Python-модулю; должен содержать хотя бы один класс `BasePlugin`. Если опущен — используется `main.py` в корне архива |
| `metainfo` | Путь к файлу метаданных YAML/JSON/Python. Если опущен — ищется `metainfo.yaml`/`metainfo.yml`/`metainfo.json` в корне |
| `assets` | Корневая директория ресурсов; если ключа нет — автоматически используется корневая папка `assets` |
| `strings` | Один файл локализации или директория локалей |
| `wheels` | Директория с бандл-`.whl` |

> 💡 Во входном модуле должен быть **один** класс плагина: инстанцируется только первый найденный `BasePlugin`-подкласс. Хелпер-классы импортируйте из других файлов.

Builder-only ключи (например, `elyxbuilder: plugin/.elyxbuilder`) рантайм игнорирует, но они доступны коду через `from elyx import refmap`.

### Правила архива

Расширения: `.elyx`, `.eaf`, `.elyx.zip`, `.eaf.zip`. Файлы проекта должны лежать **в корне** архива:

```
✅ my_plugin.elyx
├── refmap.yml        ← найден
├── metainfo.yml
└── main.py

❌ my_plugin.elyx
└── my_plugin/
    ├── refmap.yml    ← не в корне — не найден
    └── main.py
```

Директории, объявленные в `refmap`, должны существовать. Для совместимости с ZIP-проверками включайте в архив настоящие записи директорий, а не только их файлы.

### Установленный расклад (для сведения)

```
<app files>/plugins/
├── ElyxPlugins/
│   └── <plugin_id>/           ← распакованный плагин
├── elyx_local_libs/<plugin_id>/  ← извлечённые колёса
└── archives/
```

Это детали рантайма, не импорт-пути. Код плагина должен использовать окружение `elyx` и обычные локальные импорты.

## Метаданные (`metainfo`)

Структурированные плагины держат метаданные вне входного модуля. Рекомендуется YAML:

```yaml
id: better_previews
name: Better Previews
description: "{plugin_description}"
author: "@yourUsername"
version: "1.2.0"
icon: exteraPlugins/1
app_version: ">=12.5.1"
sdk_version: ">=1.4.5.3"
requirements: "requests>=2.32, PyYAML>=6"
requires:
  helper_plugin (1.4.0): https://example.com/helper_plugin.elyx
```

### Поля

| Поле | Обязателен | По умолчанию | Смысл |
|---|---|---|---|
| `id` | ✅ | — | Стабильный идентификатор |
| `name` | ✅ | — | Отображаемое имя |
| `description` | — | `Description not provided` | Описание (Markdown) |
| `author` | — | `Unknown author` | Автор или хэндл |
| `version` | — | `1.0.0` | Версия релиза |
| `icon` | — | пусто | Ссылка на иконку хоста, например `exteraPlugins/1` |
| `min_version` | — | пусто | Legacy минимальная версия приложения |
| `app_version` | — | пусто | Выражение совместимости приложения |
| `sdk_version` | — | пусто | Выражение совместимости SDK |
| `requirements` | — | пусто | Пакеты через PIP-контроллер |
| `requires` | — | `{}` | Зависимости на другие плагины |

Ключи в стиле `__name__` нормализуются в `name`, когда используется явный файл метаданных; в новых проектах используйте обычные имена.

### Правила `id`

2–32 символа, только ASCII-буквы, цифры и `_`. Дефисы, пробелы и не-ASCII **не принимаются** (строже, чем в однофайловых плагинах). Версии пишите в кавычках, чтобы парсер YAML не переинтерпретировал значения:

```yaml
version: "2.0.0-beta.1"
app_version: ">=12.5.1"
sdk_version: ">=1.4.5.3"
```

### Локализованное описание

```yaml
description: "{plugin_description}\n\n[Source code](https://example.com)"
```

Плейсхолдер `{plugin_description}` резолвится при загрузке метаданных: сначала текущая локаль, потом английский; отсутствующий ключ подставляется своим именем.

### `requirements`

Строка с запятыми:

```yaml
requirements: "requests>=2.32, tinydb>=4.8"
```

Движок парсит значение и просит хост-PIP-контроллер установить пакеты **до** импорта `main`.

### `requires` — зависимость от других плагинов

```yaml
requires:
  helper_plugin: https://example.com/helper_plugin.elyx
  shared_runtime (2.1.0): https://example.com/shared_runtime.elyx
```

- ключ маппинга — id плагина;
- в скобках — опциональная минимальная версия;
- значение — опциональная ссылка на скачивание для установщика;
- требуемый плагин должен быть установлен и включён.

### Метаданные в рантайме

```python
from elyx import metainfo

self.logger.info(f"Running {metainfo['name']} {metainfo['version']}")
```

`metainfo` — словарь с нормализованными и сгенерированными полями; недокументированные ключи считайте деталями реализации. Для опциональных окружений:

```python
from elyx import get_environment
metadata = get_environment()["metainfo"]
```

## Модули и импорты

Каждый установленный плагин получает собственное внутреннее пространство имён модулей: `ElyxPlugins.<plugin_id>.*`. Вы обычно не печатаете это имя — оно защищает `sys.modules от коллизий между плагинами с одинаковыми файлами.

### Локальные импорты

```python
# main.py
from helpers import normalize_url          # обычный импорт
from features.links import build_preview
from .helpers import normalize_url         # относительный тоже работает
from .features.links import build_preview
```

Одинаковые имена файлов в разных плагинах не конфликтуют.

### Динамические импорты

Только через публичный хелпер:

```python
from elyx import import_module

links = import_module("features.links")
```

Обычный `importlib.import_module("features.links")` не имеет контекста плагина и может обойти локальное разрешение — не используйте его для локальных модулей.

### Поддерживаемые файлы

- `.py` исходники; `.pyc` байткод; пакеты с `__init__.py`/`__init__.pyc`; namespace-директории (без `__init__.py`).

> ⚠️ Байткод должен совпадать по magic number с рантаймом: **Python 3.11**. `.pyc`, собранный другой версией Python, отвергается как несовместимый. Байткод — не граница безопасности и не защита секретов.

### Импорт данных как модулей

JSON, YAML, YML и TXT-файлы импортируются, если их stem — валидный путь модуля:

```python
import config            # config.json

self.logger.info(config.api_url)
self.logger.info(str(config["limits"].page_size))
self.logger.info(str(config.get("missing", "fallback")))

import notice            # notice.txt
self.logger.info(notice.content)
```

### Зарезервированные имена

Импортер оставляет SDK- и Java-корни обычному рантайму: `android`, `androidx`, `base_plugin`, `client_utils`, `com`, `de`, `elyx`, `hook_utils`, `importlib`, `java`, `org`, `ui`. **Не создавайте локальные пакеты с этими именами.** Также не теньте стандартную библиотеку (`json`, `pathlib`, `typing`).

### Поведение перезагрузки

При reload/unload плагина его namespaced-модули удаляются, кеши путей сбрасываются — следующая загрузка исполняет свежий код. Но изоляция покрывает только модули Python: глобальное состояние (потоки, сокеты, наблюдатели, мутации сторонних модулей) чистите вручную в `on_plugin_unload`.

## Ресурсы (Assets)

Объявите директорию в `refmap.yml` (`assets: plugin/res`), затем:

```python
from elyx import assets
```

`assets` существует только при валидной директории ресурсов. Если ресурсы опциональны:

```python
from elyx import get_environment
assets = get_environment().get("assets")
```

### Доступ к ресурсу

Три эквивалентных стиля + вложенность:

```python
logo = assets.logo
logo = assets["logo"]
logo = assets.get("logo")
logo = assets["logo.svg"]          # можно с расширением
add_icon = assets["icons/add"]     # вложенные пути
add_icon = assets.icons.add
```

Атрибутный доступ использует нормализованный stem: не-ASCII-буквы и цифры → `_` (`empty-state.svg` → `assets.empty_state`; `logo.dark.png` → `assets.logo`). Избегайте неоднозначных stem: `logo.svg` и `logo.png` имеют одинаковый stem `logo`.

### `Assets` и `Asset`

```python
len(assets)              # число прямых детей
"logo" in assets         # проверка наличия
for filename in assets:  # итерация
    ...
assets.parent            # родительская директория
assets.dir_path          # pathlib.Path
```

`Asset`:

| Член | Смысл |
|---|---|
| `name` | Нормализованное логическое имя |
| `filename` | Реальное имя файла |
| `ext` | Последнее расширение без точки |
| `path` / `path_str` | `pathlib.Path` / строка |
| `java_file` | `java.io.File` |

### Чтение содержимого

```python
raw_bytes = assets.banner.content_bytes()
text = assets.about.content_string()
json_data = assets.loading.content_json()
yaml_data = assets.defaults.content_yaml()
automatic = assets.defaults.content()   # json/yaml → dict; иначе UTF-8 текст; иначе bytes
```

### Android-конвертация

```python
drawable = assets.banner.to_drawable()                    # Drawable.createFromPath
location = assets.banner.to_image_location()              # Telegram ImageLocation
bmp = assets.banner.to_bitmap_drawable(width=64, height=64)

svg_drawable = assets.logo.to_svg_drawable()              # SVG
scaled = assets.logo.to_svg_drawable(width=48, height=48)
svg_bitmap = assets.logo.to_svg_bitmap(width=48, height=48, white=True)

thumb = assets.logo.to_svg_thumb(Theme.key_windowBackgroundWhiteBlueIcon, 1.0)  # тематическая

animation = assets.loading.to_lottie_drawable(width=32, height=32)  # Lottie (dp)
```

### Создание ресурса из другого пути

```python
from elyx import Asset

asset = Asset.from_path("/absolute/path/to/file.svg")  # также pathlib.Path и java.io.File

# Временный скачанный ресурс (синхронная сеть и I/O — только из фоновой очереди!):
remote = Asset.temp_asset_from_url("https://example.com/icon.svg", "downloaded_icon.svg")
```

### Ошибки

- `AssetNotFoundException` — файла нет;
- `AssetsDirNotFoundException` — корень ресурсов исчез;
- `ValueError` — путь вне разрешённых директорий ресурсов.

Бандл-пути для плагина read-only; изменяемые пользовательские данные храните через хелперы app data/cache ([08. Файлы](08-files-intents-reflection.md)).

## Локализация (strings)

Объявите путь в `refmap.yml` (`strings: plugin/locales`):

```
plugin/locales/
├── strings_en.yml
├── strings_ru.yml
└── strings_de.json
```

Локаль — часть после последнего `_` в stem файла: `strings_en.yml` → `en`, `ui_de.yaml` → `de`, `strings.yml` → `en`. Форматы: `.yaml`, `.yml`, `.json`, `.py` (рекомендуются YAML/JSON; `.py` исполняется при загрузке, собираются простые публичные неколлбэк-значения).

### Чтение

```python
from elyx import strings

title = strings["title"]
title = strings.title
title = strings.get("title")
title = strings("title")
```

Вызов объекта ещё и форматирует:

```python
message = strings("hello", name="Alice")     # "Hello, {name}!"
downloaded = strings("downloaded", count=5)
text = strings("coordinates", 10, 20)        # позиционные: "Position: {}, {}"
```

### Порядок fallback

1. выбранная локаль плагина;
2. английский (`en`);
3. сам ключ:

```python
label = strings("missing_key")   # "missing_key", если нигде не определён
label = strings.get("optional_label", "Fallback label")
label = strings("optional_label", default="Fallback label")
```

Явная локаль: `strings.get_with_locale("title", locale="ru")` или `strings("title", locale="en")`. Выбранная локаль берётся из настройки языка Elyx; когда она пуста — системная локаль Android.

### Плюрализация

`Strings.pluralize` использует три славянских формы:

```yaml
file_forms: [файл, файла, файлов]
```

```python
text = strings.pluralize(21, "file_forms")   # "21 файл"
```

Для языков с иными правилами выбирайте форму в коде или храните готовые фразы — текущий хелпер не полноценный CLDR-движок.

### Лучшие практики

- всегда включайте английский как fallback;
- стабильные семантические ключи (`settings_privacy_title`);
- не собирайте предложения конкатенацией переводных фрагментов;
- передавайте значения через `{named}`-плейсхолдеры;
- кавычки в YAML вокруг текста с `:`, `{}` или ведущими спецсимволами.

## Хранилище настроек (settings)

Каждый Elyx-плагин получает `SettingsController`, привязанный к его `id`:

```python
from elyx import settings
```

Хранит значения в той же хост-системе, что и `BasePlugin.get_setting` и строки `ui.settings`.

```python
enabled = settings.get_setting("enabled", True)
enabled = settings.get("enabled", True)     # короткий алиас
enabled = settings("enabled", True)         # вызов
enabled = settings["enabled"]               # без явного default!
settings.set_setting("enabled", True)
settings.set("page_size", 50)
settings["username"] = "Alice"
settings.set("mode", "advanced", reload_settings=True)  # перестроить открытый экран
values = settings.get_settings()            # все значения
settings.clear_settings()                   # удалить все значения плагина
```

> `settings["enabled"]` не имеет явного default — предпочитайте `get`, когда fallback важен. `reload_settings=True` — только когда от значения зависит видимость/структура строк.

Ключ строки настроек UI — тот же, что виден через `settings`:

```python
from base_plugin import BasePlugin
from elyx import settings, strings
from ui.settings import Header, Input, Switch
from typing import Any, List


class SettingsExamplePlugin(BasePlugin):
    def create_settings(self) -> List[Any]:
        return [
            Header(text=strings("settings_title")),
            Switch(key="enabled", text=strings("enabled"), default=True),
            Input(key="username", text=strings("username"), default="Alice"),
        ]

    def on_plugin_load(self):
        if settings.get("enabled", True):
            self.logger.info(f"Hello {settings.get('username', 'Alice')}")
```

**Что хранить в настройках:** булевы, числа, короткие строки, маленькие списки, UI-предпочтения. **Что хранить в файлах/БД:** большие JSON, кеши, скачанные медиа, часто меняющаяся история, секреты.

## Зависимости

Три механизма — выбирайте наименьший подходящий:

### 1. PIP-пакеты (`requirements`)

```yaml
requirements: "requests>=2.32, tinydb>=4.8"
```

До импорта входного модуля Elyx просит хост-PIP-контроллер установить пакеты и добавляет их пути в `sys.path`. Ограничения те же, что у обычного PIP-механизма: только pure-Python колёса, никакой компиляции под Android. Колесо, собранное под Windows/macOS/десктоп-Linux, на телефоне работать не будет.

### 2. Бандл-колёса (`wheels`)

```yaml
wheels: wheels
```

```
wheels/
├── my_library-1.0.0-py3-none-any.whl
└── another_library-2.1.0-py3-none-any.whl
```

При загрузке Elyx: находит прямые `.whl`-дети директории → извлекает в плагин-специфичное локальное хранилище → пропускает уже извлечённое с тем же stem → удаляет устаревшие извлечённые директории → добавляет пути импорта.

Полезно, когда pure-Python библиотека не публикуется в индекс, артефакт должен ехать с плагином или установка не должна зависеть от сети. **При изменении содержимого колеса пересоберите/переименуйте его** — Elyx идентифицирует извлечённую версию по stem имени файла.

### 3. Локальные source-пакеты

Малым библиотекам колесо не нужно:

```
vendor/
├── __init__.py
└── parser.py
```

```python
from vendor.parser import parse
```

Локальный исходник namespaced на плагин и участвует в live reload.

### Обязательные плагины (`requires`)

Декларация проверяет наличие; она не создаёт стабильный Python API между плагинами. Если плагины общаются — определите явный контракт совместимости и не импортируйте приватные namespaced-модули другого плагина.

### Версии и безопасность

- PIP-зависимости ассоциированы с id плагина; удаление плагина просит хост удалить его PIP-зависимости; смена `id` создаёт отдельный скоуп зависимостей.
- Считайте зависимости исполняемым кодом: пинуйте версии, распространяйте только проверенные колёса, соблюдайте лицензии, не кладите креденшелы в архив. Elyx отвергает path traversal при извлечении, но это не делает недоверенную зависимость безопасной.

## Стабильное публичное API `elyx`

```python
from elyx import (
    Asset, AssetNotFoundException, Assets, AssetsDirNotFoundException,
    Callback, Callback2, Callback3, CallbackReturn, LazyDict,
    OnClickListener, Runnable, SettingsController, Strings,
    gen, gen2, get_environment, import_module, mvel_execute,
)
```

Динамические плагин-привязанные значения (`assets`, `metainfo`, `refmap`, `settings`, `strings`) берутся явными импортами и в `__all__` не входят.

### Java-колбэк-прокси

| Класс | Java-интерфейс | Метод | Возвращает значение колбэка |
|---|---|---|---|
| `OnClickListener` | `android.view.View.OnClickListener` | `onClick` | нет |
| `Runnable` | `java.lang.Runnable` | `run` | нет |
| `Callback` | `Utilities.Callback` | `run` | нет |
| `Callback2` | `Utilities.Callback2` | `run` | нет |
| `Callback3` | `Utilities.Callback3` | `run` | нет |
| `CallbackReturn` | `Utilities.CallbackReturn` | `run` | да |

```python
from elyx import OnClickListener, Runnable

button.setOnClickListener(OnClickListener(lambda view: self.logger.info("Clicked")))
queue.postRunnable(Runnable(lambda: self.logger.info("Running")))

# Дополнительные аргументы конструктора добавляются к Java-аргументам:
def on_click(view, item_id):
    self.logger.info(f"Clicked {item_id}")
button.setOnClickListener(OnClickListener(on_click, "save"))
```

Исключения ловятся и пишутся в лог приложения, не пересекая границу Java-колбэка.

### Генераторы прокси

```python
from android.content import DialogInterface
from elyx import gen, gen2

# Однометодный интерфейс:
OnDismiss = gen(DialogInterface.OnDismissListener, "onDismiss")
dialog.setOnDismissListener(OnDismiss(lambda dialog: self.logger.info("Dismissed")))

# С возвращаемым значением (default_value — если колбэк упал):
Predicate = gen(SomeJavaPredicate, "test", return_value=True, default_value=False)

# Многометодный интерфейс:
Listener = gen2(
    SomeJavaListener,
    onStart=lambda value: log_start(value),
    onFinish=lambda value: log_finish(value),
)
instance = Listener()
```

### MVEL из Python

```python
from elyx import mvel_execute

visible = mvel_execute("enabled && count > 0", {"enabled": True, "count": 3})
```

Компилирует и кеширует MVEL-выражение, конвертирует Python-dict в `java.util.HashMap`, исполняет. Опции: `to_type` — конвертация результата в Java-тип; `java_instance` — значение `this`. `data` — `dict` или `HashMap`.

### `LazyDict`

Словарь с атрибутным доступом:

```python
from elyx import LazyDict
value = LazyDict({"title": "Hello"})
value.title  # "Hello"; отсутствующий атрибут → KeyError
```

## Ошибки при доступе к окружению

`from elyx import assets` падает → путь `assets` отсутствует, пуст или невалиден. `from elyx import strings` падает → локализации не загрузились (путь/формат/нет ни одного ключа). `get_environment()` вне Elyx-кода → `RuntimeError`. Подробности — в [17. Troubleshooting](17-troubleshooting.md).
