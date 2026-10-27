# 02. Анатомия плагина: метаданные, класс, жизненный цикл

Из чего состоит файл плагина и как он живёт внутри приложения.

## Обязательная структура

Каждый файл плагина содержит:

1. **метаданные** — обычные константы верхнего уровня;
2. **один класс**, наследующийся от `BasePlugin`.

```python
from base_plugin import BasePlugin

__id__ = "weather"
__name__ = "Weather"


class WeatherPlugin(BasePlugin):
    pass
```

> ⚠️ Метаданные парсятся из исходника через **AST**. Они должны быть простыми литералами верхнего уровня — не функциями, не выражениями, не вычисляемыми значениями.

## Метаданные

```python
__id__ = "better_previews"
__name__ = "Better Previews"
__description__ = "Modifies specific URLs for better previews"
__author__ = "@immat0x1"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
__requirements__ = ["mpmath", "tinydb"]
```

### Поддерживаемые ключи

| Ключ | Обязателен | Описание |
|---|---|---|
| `__id__` | ✅ | Стабильный идентификатор плагина |
| `__name__` | ✅ | Отображаемое имя |
| `__description__` | — | Описание; поддерживает базовый Markdown в UI |
| `__author__` | — | Текст или Telegram-хэндл вида `@yourUsername` |
| `__version__` | — | По умолчанию `1.0` |
| `__icon__` | — | Формат `СтикерПак/индекс`, например `exteraPlugins/1` |
| `__app_version__` | — | Ограничение версии приложения: `>=`, `<=`, `==`, `>`, `<` |
| `__sdk_version__` | — | Минимальная версия SDK, если плагин использует новые API |
| `__requirements__` | — | Список PyPI-пакетов для установки (см. [13. PIP и зависимости](13-dependencies.md)) |
| `__priority__` | — | Порядок загрузки плагинов: меньше = раньше (не описан в официальных доках, но поддерживается движком; библиотеки типа moonlib ставят `__priority__ = 1`, чтобы грузиться до потребителей) |

**Совместимость:** `__min_version__` всё ещё принимается, но трактуется как `__app_version__ = ">=..."`. В новых плагинах используйте `__app_version__`.

### Правила `__id__`

- длина **2–32 символа**;
- начинается **с буквы**;
- только латинские буквы, цифры, `_` и `-`.

`id` владеет папкой плагина, настройками, зависимостями, состоянием включённости, хуками и пунктами меню. **Смена `id` создаёт новый плагин**, а не обновляет старый.

### Практический baseline

Заявляйте те версии, на которых реально тестировали:

```python
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"
```

Если используете account-хелперы (`self.client()`, `get_client()`, `account=`) — заявляйте `__sdk_version__ = ">=1.4.5.0"`, чтобы старые установки падали с понятным сообщением, а не с `TypeError`.

## Жизненный цикл

### Загрузка и выгрузка

```python
from base_plugin import BasePlugin


class DebugPlugin(BasePlugin):
    def on_plugin_load(self):
        # Регистрируйте хуки, слушатели, фоновые задачи здесь.
        self.logger.info("Plugin loaded!")

    def on_plugin_unload(self):
        # Остановите то, что не убирается автоматически: потоки, сокеты, наблюдатели.
        self.logger.info("Plugin unloaded!")
```

- `on_plugin_load` — при включении плагина или восстановлении при старте приложения.
- `on_plugin_unload` — при выключении плагина или завершении работы приложения.

> Java-хуки и пункты меню убираются автоматически. **Потоки, сокеты, наблюдатели и собственные слушатели — ваша забота.**

### События приложения

```python
from base_plugin import AppEvent, BasePlugin


class DebugPlugin(BasePlugin):
    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.START:
            self.logger.info("App is starting")
        elif event_type == AppEvent.STOP:
            self.logger.info("App is stopping")
        elif event_type == AppEvent.PAUSE:
            self.logger.info("App moved to background")
        elif event_type == AppEvent.RESUME:
            self.logger.info("App returned to foreground")
```

Значения `AppEvent`: `START`, `STOP`, `PAUSE`, `RESUME`.

## Хранилище настроек

`BasePlugin` уже содержит хелперы для постоянных значений:

| Метод | Описание |
|---|---|
| `self.get_setting(key, default=None)` | Прочитать значение |
| `self.set_setting(key, value, reload_settings=False)` | Записать; `reload_settings=True` перестраивает открытый экран настроек |
| `self.export_settings()` | Экспортировать все настройки (dict) |
| `self.import_settings(settings, reload_settings=True)` | Импортировать словарь настроек |

```python
current_value = self.get_setting("enabled", True)
self.set_setting("enabled", not current_value, reload_settings=True)

backup = self.export_settings()
self.import_settings(backup, reload_settings=False)
```

Подробности про экран настроек — в [03. Настройки плагина](03-settings.md).

## Логирование

`self.logger` пишет в logcat под тегом `plugin.<id>`:

```python
self.logger.debug("Hook installed for %s", class_name)

try:
    self.process(update)
except Exception:
    self.logger.exception("Failed to process update")

self.logger.dump(message.messageOwner)
```

Просмотр:

```bash
adb logcat -s plugin.my_plugin
```

Устаревший `self.log(message)` работает, но это legacy. Полный справочник — в [07. Android-утилиты и логирование](07-android-utils-and-logging.md).

## Пункты меню (Menu Items)

Плагин может добавлять пункты в несколько меню приложения через `MenuItemData`.

```python
from typing import Any, Dict

from base_plugin import BasePlugin, MenuItemData, MenuItemType


class MyMenuPlugin(BasePlugin):
    def on_plugin_load(self):
        self.message_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.MESSAGE_CONTEXT_MENU,
                text="Log Message Info",
                on_click=self.handle_message_click,
                icon="msg_info",
                subtext="Debug helper",
                priority=10,
            )
        )

        self.profile_item_id = self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.PROFILE_ACTION_MENU,
                text="Log User Info",
                on_click=self.handle_profile_click,
                icon="user_search",
            )
        )

    def handle_message_click(self, context: Dict[str, Any]):
        self.logger.info(f"Menu item clicked. Context keys: {list(context.keys())}")
        message = context.get("message")
        if message:
            self.logger.info(f"Message ID: {message.getId()}")

    def handle_profile_click(self, context: Dict[str, Any]):
        user = context.get("user")
        if user:
            self.logger.info(f"Profile: {user.first_name} (ID: {user.id})")
```

### Поля `MenuItemData`

| Поле | Тип | Описание |
|---|---|---|
| `menu_type` | `MenuItemType` | Обязательно. Целевое меню |
| `text` | `str` | Обязательно. Видимый заголовок |
| `on_click` | `Callable[[Dict], None]` | Обязательно. Обработчик нажатия |
| `item_id` | `Optional[str]` | Стабильный id для последующего удаления |
| `icon` | `Optional[str]` | Имя drawable, например `"msg_info"` |
| `subtext` | `Optional[str]` | Вторая строка под заголовком |
| `condition` | `Optional[str]` | MVEL-выражение, управляющее видимостью |
| `priority` | `int` | Большие значения обычно показываются выше |

### Типы меню `MenuItemType`

- `MESSAGE_CONTEXT_MENU` — контекстное меню сообщения;
- `DRAWER_MENU` — боковая панель;
- `MAIN_MENU` — главное меню;
- `CHAT_ACTION_MENU` — меню действий в чате;
- `PROFILE_ACTION_MENU` — меню действий в профиле.

### Контекст `on_click`

Обработчик получает словарь; набор ключей зависит от типа меню и экрана. Частые ключи: `account`, `context`, `fragment`, `dialog_id`, `user`, `userId`, `userFull`, `chat`, `chatId`, `chatFull`, `encryptedChat`, `message`, `groupedMessages`, `botInfo`.

Надёжнее всего во время разработки вывести ключи в лог:

```python
def handle_click(self, context):
    self.logger.info(f"Available keys: {list(context.keys())}")
```

### Удаление пунктов

При выгрузке плагина пункты убираются автоматически. Вручную:

```python
if self.message_item_id:
    self.remove_menu_item(self.message_item_id)
```

## Куда дальше

- [03. Настройки плагина](03-settings.md) — полный UI-конструктор настроек
- [04. Хуки событий](04-hooks.md) — перехват запросов, обновлений, исходящих сообщений
