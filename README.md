# 📚 Документация плагинов ExteraGram

Полное руководство по разработке плагинов для [exteraGram](https://exteragram.app/) — модифицированного Android-клиента Telegram с движком плагинов на Python.

> Основано на официальной документации SDK (plugins.exteragram.app/docs, SDK 1.4.4.3) и дополнено разбором инструментов и библиотек сообщества: ElyxBuilder, AltyLib, PACLibs и примеров из 50+ реальных плагинов.

## Быстрый старт

Плагин — это Python-файл с метаданными и одним классом:

```python
from base_plugin import BasePlugin

__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "You"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"


class HelloWorldPlugin(BasePlugin):
    def on_plugin_load(self):
        self.logger.info("Hello!")
```

Установите свежий exteraGram (12.5.1+), включите движок плагинов и developer mode — и вперёд: [01. Установка и начало работы](docs/01-getting-started.md).

## Оглавление

### Основы

| # | Раздел | Содержание |
|---|--------|-----------|
| 01 | [Установка и начало работы](docs/01-getting-started.md) | Приложение, движок плагинов, окружение, ADB, отладка VS Code |
| 02 | [Анатомия плагина](docs/02-plugin-anatomy.md) | Метаданные, `BasePlugin`, жизненный цикл, `AppEvent`, настройки-хелперы, пункты меню |
| 03 | [Настройки плагина](docs/03-settings.md) | `ui.settings`: Switch/Selector/Input/EditText, `Custom`, `SimpleSettingFactory` |
| 04 | [Хуки событий](docs/04-hooks.md) | Перехват запросов, ответов, updates, исходящих сообщений; `HookResult`/`HookStrategy` |

### SDK-справочник

| # | Раздел | Содержание |
|---|--------|-----------|
| 05 | [Клиентские утилиты](docs/05-client-utils.md) | Очереди, TL-запросы, `send_*`, `edit_message`, контроллеры, NotificationCenter, **мультиаккаунт** |
| 06 | [UI: диалоги и уведомления](docs/06-ui-dialogs-and-notifications.md) | `AlertDialogBuilder`, `BulletinHelper`, `android_utils` (UI-поток, клипборд) |
| 07 | [Логирование](docs/07-android-utils-and-logging.md) | `log_utils`: уровни, теги, `dump()`, таймеры, хендлеры, миграция с `log()` |
| 08 | [Файлы, Intents, рефлексия](docs/08-files-intents-reflection.md) | `file_utils`, `FilesController`, `IntentsManager` (deeplink-хуки), `hook_utils` |
| 09 | [Xposed-хуки](docs/09-xposed-hooking.md) | `MethodHook`/`MethodReplacement`, фильтры `HookFilter`, практические примеры |
| 10 | [Class Proxy](docs/10-class-proxy.md) | Java-подклассы из Python: DSL `java_subclass`/`joverride`/`jfield`, MVEL, модификаторы `JA`/`JS`/`JIR` |
| 13 | [PIP и зависимости](docs/13-dependencies.md) | `__requirements__`, pure-Python ограничения, предустановленные библиотеки |

### Формат Elyx и инструменты

| # | Раздел | Содержание |
|---|--------|-----------|
| 11 | [Elyx: структурированные плагины](docs/11-elyx-structured-plugins.md) | Многофайловые плагины: `refmap`, `metainfo`, модули, assets, локализация, wheels, публичное API `elyx` |
| 12 | [ElyxBuilder и DevServer](docs/12-elyxbuilder-and-dev-server.md) | CLI `elyb`, watch, сборка/компиляция/шифрование, протокол DevServer (порт 42690), live reload, remote debug |

### Экосистема

| # | Раздел | Содержание |
|---|--------|-----------|
| 14 | [Библиотеки сообщества](docs/14-community-libraries.md) | Механика библиотек-плагинов (sys.modules), zwylib/mandre_lib/excomponents, PACLibs, AltyLib, DEX-техники с рисками |
| 15 | [Примеры и рецепты](docs/15-examples-and-recipes.md) | Приёмы из реальных плагинов: разбор 775 плагинов магазина, статистика API, DEX-bridge, TL-запросы |
| 16 | [Публикация](docs/16-publishing.md) | Чеклист релиза, безопасность, аудит, площадки распространения |
| 17 | [FAQ и Troubleshooting](docs/17-troubleshooting.md) | Диагностика ошибок по всем модулям SDK и Elyx |

## Ключевые факты

- **Язык плагинов:** Python 3.11 внутри приложения (Chaquopy v16).
- **Базовые версии:** SDK 1.4.4.3, exteraGram 12.5.1+ (account-хелперы — SDK 1.4.5.0+).
- **Куда класть плагин:** `/data/user/0/com.exteragram.messenger/files/plugins/<id>.py`.
- **DevServer:** `127.0.0.1:42690` через `adb forward tcp:42690 tcp:42690`; отладчик — отдельный порт (debugpy, платформы `vscode`/`pycharm`).
- **Зависимости:** только pure-Python пакеты с PyPI (`-none-any.whl`); бинарные расширения (numpy и т.п.) не поддерживаются.
- **Предустановлено:** requests, pillow, beautifulsoup4, lxml, PyYAML, packaging, debugpy.

## Что умеет SDK

- жизненный цикл плагина (`BasePlugin`: load/unload/app events);
- перехват TL-запросов, ответов, updates и исходящих сообщений (`HookResult`/`HookStrategy`);
- Xposed-хуки Java-методов и конструкторов + генерация Java-подклассов (Class Proxy, DexMaker);
- экраны настроек в стиле Telegram + пункты в 5 меню приложения;
- отправка/редактирование сообщений и медиа, диалоги, буллетины, deeplink-хуки;
- перехват открытия файлов по расширению;
- Elyx: многофайловые плагины с ресурсами, локализацией и live reload.

## Ссылки

| Ресурс | URL |
|---|---|
| Официальная документация SDK | https://plugins.exteragram.app/docs |
| exteraSquad (SDK/приложение) | https://github.com/exteraSquad |
| ElyxBuilder | https://github.com/shareui/ElyxBuilder |
| Коллекция плагинов | https://github.com/fan-world-me/extragram_plugins |
| awesome-telegram-plugins | https://github.com/makarworld/awesome-telegram-plugins |
| AltyLib | https://github.com/Altuskhins/AltyLib |
| PACLibs | https://github.com/PacHub228/PACLibs |
| Chaquopy (Python для Android) | https://github.com/chaquo/chaquopy |
| Схема TL-методов Telegram | https://corefork.telegram.org/schema |

---

*Документация дополняется разборами реальных плагинов — раздел [15](docs/15-examples-and-recipes.md) будет пополняться.*
