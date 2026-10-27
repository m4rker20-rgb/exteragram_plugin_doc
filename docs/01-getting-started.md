# 01. Установка и начало работы

Как подготовить окружение и запустить первый плагин exteraGram.

## Что такое плагины exteraGram

Плагины exteraGram — это Python-код, который выполняется прямо внутри приложения через встроенный движок плагинов. SDK построен на **Chaquopy v16** (Python ↔ Java interop) и поддерживает Xposed-подобное перехватывание Java-методов.

**Актуальный baseline SDK:**

| Параметр | Значение |
|---|---|
| Версия SDK | `1.4.4.3` (account-хелперы — с `1.4.5.0`) |
| Python runtime | **3.11** |
| Минимальная версия приложения | **exteraGram 12.5.1+** |

> ⚠️ **Версия приложения важна.** Если сборка приложения слишком старая, SDK уходит в safe mode и модули плагинов отказываются инициализироваться.

## Шаг 1. Установите свежую сборку exteraGram

- Скачайте актуальный билд (тестовые сборки обычно публикуются в **beta-канале**).
- Убедитесь, что версия **12.5.1 или новее**.

## Шаг 2. Включите движок плагинов

1. Зайдите в **exteraGram Preferences → Plugins** и включите движок плагинов.
2. Откройте информационный экран плагинов и включите **developer mode** (режим разработчика).
   Он нужен для DevServer, отладки и debug-логов.

## Шаг 3. Создайте проект на ПК

Создайте папку и положите в неё Python-файл плагина:

```
my-plugin/
└── hello_world.py
```

Рекомендации:

- имя файла должно совпадать с `__id__` плагина: `hello_world.py` ↔ `__id__ = "hello_world"`;
- держите метаданные **обычными константами верхнего уровня** — загрузчик парсит их через AST, динамические значения не читаются;
- для автодополнения в IDE укажите путь к папке `src/` SDK-репозитория (или поставьте community-пакет `pip install exteragram-utils` — типизированные заглушки для pyright/VS Code; ошибки импортов pyright при этом нормальны).

### Минимальный файл плагина

```python
from base_plugin import BasePlugin

__id__ = "hello_world"
__name__ = "Hello World"
__description__ = "My first exteraGram plugin"
__author__ = "Your Name"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"


class HelloWorldPlugin(BasePlugin):
    pass
```

## Шаг 4. Подключитесь к устройству

Подключите телефон к ПК по USB (или используйте эмулятор) и убедитесь, что `adb` доступен в `PATH`.

Два рабочих сценария:

1. **Ручной** — скопировать `.py`-файл в папку плагинов приложения и перезагрузить плагин вручную:

   ```
   adb push hello_world.py /data/user/0/com.exteragram.messenger/files/plugins/hello_world.py
   ```

2. **DevServer** — использовать встроенный TCP-сервер приложения (порт по умолчанию `42690`) для записи/перезагрузки/отладки:

   ```bash
   adb forward tcp:42690 tcp:42690
   ```

Подробности протокола и удалённая отладка — в разделе [12. ElyxBuilder и DevServer](12-elyxbuilder-and-dev-server.md).

### Настройка VS Code для отладки

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python Debugger: Attach exteraGram plugin",
            "type": "debugpy",
            "request": "attach",
            "connect": { "host": "localhost", "port": 5678 },
            "pathMappings": [
                {
                    "localRoot": "/path/to/your/workspace",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
                }
            ]
        }
    ]
}
```

Файл плагина на устройстве тоже должен называться по id: `hello_world.py`. Отладочный порт (`5678`) — отдельный от порта DevServer (`42690`), его выбираете вы или ваш инструмент.

## Что читать дальше

1. [02. Анатомия плагина](02-plugin-anatomy.md) — метаданные, жизненный цикл, меню
2. [03. Настройки плагина](03-settings.md) — экран настроек
3. [04. Хуки событий](04-hooks.md) — перехват запросов, обновлений и исходящих сообщений
4. [05. Клиентские утилиты](05-client-utils.md) — очереди, отправка сообщений, контроллеры

## Полезные ресурсы

| Ресурс | Ссылка |
|---|---|
| Официальная документация SDK | https://plugins.exteragram.app/docs |
| Репозиторий SDK | https://github.com/exteraSquad |
| Инструмент сборки ElyxBuilder | https://github.com/shareui/ElyxBuilder |
| Коллекция плагинов | https://github.com/fan-world-me/extragram_plugins |
| awesome-telegram-plugins | https://github.com/makarworld/awesome-telegram-plugins |
| Telegram-каналы плагинов | exteraGram Plugins (официальный), Dotted (@exteraDev), zwyPlugins (@zwylair) |
