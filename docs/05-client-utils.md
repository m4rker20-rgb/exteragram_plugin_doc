# 05. Клиентские утилиты (`client_utils`)

Очереди, Telegram-запросы, отправка/редактирование сообщений, контроллеры и NotificationCenter.

Модуль `client_utils` — высокоуровневые помощники для взаимодействия с внутренностями Telegram без постоянного погружения в Java.

Типовые задачи:

- постановка работы в очередь вне UI-потока;
- отправка Telegram-запросов;
- отправка и редактирование сообщений и медиа;
- доступ к текущему фрагменту и контроллерам аккаунта;
- подписка на `NotificationCenter`.

## Очереди (фоновые потоки)

`run_on_queue(...)` — для работы, которая не должна блокировать UI-поток: сетевые запросы, файловый I/O, тяжёлый парсинг, длинные вычисления.

```python
import time

from android_utils import log
from client_utils import run_on_queue


def my_long_task(parameter: str):
    log(f"Task started with: {parameter}")
    time.sleep(5)
    log(f"Task finished for: {parameter}")


# На дефолтной очереди PLUGINS_QUEUE:
run_on_queue(lambda: my_long_task("some_data"))
```

Можно выбрать конкретную очередь и добавить задержку в миллисекундах:

```python
from client_utils import GLOBAL_QUEUE, run_on_queue

# На GLOBAL_QUEUE через 2,5 секунды:
run_on_queue(lambda: my_long_task("other_data"), GLOBAL_QUEUE, 2500)
```

Доступные константы очередей:

```python
STAGE_QUEUE = "stageQueue"
GLOBAL_QUEUE = "globalQueue"
CACHE_CLEAR_QUEUE = "cacheClearQueue"
SEARCH_QUEUE = "searchQueue"
PHONE_BOOK_QUEUE = "phoneBookQueue"
THEME_QUEUE = "themeQueue"
EXTERNAL_NETWORK_QUEUE = "externalNetworkQueue"
PLUGINS_QUEUE = "pluginsQueue"
```

Если нужен сырой Java `DispatchQueue`:

```python
from client_utils import PLUGINS_QUEUE, get_queue_by_name

plugins_dispatch_queue = get_queue_by_name(PLUGINS_QUEUE)
if plugins_dispatch_queue:
    ...  # методы DispatchQueue доступны напрямую
```

## Telegram-запросы

`send_request(request, fn)` отправляет сырой `TLObject` через `ConnectionsManager` аккаунта, выбранного в UI (или `account=`). Обычный Python-колбэк SDK оборачивает в `RequestDelegate` сам. Колбэк выполняется с аккаунтом в scope — хелперы внутри него последуют за аккаунтом (см. мультиаккаунт ниже).

```python
from java.lang import Integer
from org.telegram.tgnet import TLRPC

from android_utils import log
from client_utils import send_request


def handle_read_contents_response(response, error):
    if error:
        log(f"Error: {error.text}")
        return

    if response and isinstance(response, TLRPC.TL_messages_affectedMessages):
        log(f"Read contents. PTS: {response.pts}, Count: {response.pts_count}")
    else:
        log(f"Unexpected response type: {type(response)}")


req = TLRPC.TL_messages_readMessageContents()
req.id.add(Integer(12345))

request_id = send_request(req, handle_read_contents_response)  # → id запроса ConnectionsManager
```

## Отправка сообщений и медиа

Хелперы `send_*` сами строят `SendMessageParams`, готовят медиа-объекты и отправляют операцию в правильном потоке. Все принимают опциональный `parse_mode="HTML"` или `parse_mode="Markdown"` (регистронезависимо; парсится `message` или `caption`, заполняются `entities`).

### `send_text`

```python
from client_utils import send_text

peer_id = 123456789

send_text(peer_id, "Hello from my plugin!")                       # просто текст
send_text(peer_id, "This is a reply.", replyToMsg=9876)           # ответ
send_text(peer_id, "**Hello** from my *plugin*!", parse_mode="Markdown")
```

### `send_photo`

```python
from client_utils import send_photo

send_photo(peer_id, "/path/to/image.jpg", caption="Here is a photo!")
send_photo(
    peer_id, "/path/to/image.jpg",
    caption="<b>High quality</b> photo.",
    high_quality=True, parse_mode="HTML",
)
```

### `send_document` / `send_video` / `send_audio`

```python
from client_utils import send_audio, send_document, send_video

send_document(peer_id, "/path/to/file.zip", caption="Here is the zip file.")

# SDK заполнит стандартные атрибуты видео (размер, длительность):
send_video(peer_id, "/path/to/video.mp4", caption="Check out this video!")

# SDK заполнит аудио-метаданные, когда это возможно:
send_audio(peer_id, "/path/to/song.mp3", caption="Listen to this!")
```

### Низкоуровневый `send_message`

Для большего контроля — `send_message(params, parse_mode=None)`. Принимает словарь полей, мапящихся на `SendMessagesHelper.SendMessageParams`:

```python
from client_utils import send_message

send_message({
    "peer": 123456789,
    "message": "Scheduled message from plugin",
    "scheduleDate": 1735689600,
    "notify": False,
})
```

Часто используемые ключи: `peer`, `message`, `caption`, `photo`, `document`, `path`, `replyToMsg`, `replyMarkup`, `params`, `notify`, `scheduleDate`, `ttl`, `hasMediaSpoilers`, `sendingHighQuality`.

## Редактирование сообщений

`edit_message(...)` редактирует существующий `MessageObject`: текст, замена медиа, спойлер на заменённом медиа.

```python
from client_utils import edit_message

# message_obj — реальный org.telegram.messenger.MessageObject

edit_message(message_obj, text="This is the new, edited text.")
edit_message(message_obj, text="This is <b>bold</b> text.", parse_mode="HTML")

# Заменить медиа и обновить текст одновременно:
edit_message(message_obj, file_path="/path/to/another/image.jpg", text="Here is a new photo instead.")

# Заменить медиа со спойлером:
edit_message(message_obj, file_path=new_photo_path, with_spoiler=True)
```

- `text=` — заменить текст сообщения/подписи;
- `file_path=` — заменить прикреплённое медиа;
- неподдерживаемый `parse_mode` вызывает `ValueError`.

## Фрагменты, контроллеры и хелперы аккаунта

Возвращают компоненты Telegram для аккаунта, выбранного в UI, если явно не передан `account=` (см. [Мультиаккаунт](#мультиаккаунт)).

```python
from client_utils import (
    get_last_fragment, get_account_instance, get_messages_controller,
    get_contacts_controller, get_media_data_controller, get_connections_manager,
    get_location_controller, get_notifications_controller, get_messages_storage,
    get_send_messages_helper, get_file_loader, get_secret_chat_helper,
    get_download_controller, get_notifications_settings, get_notification_center,
    get_media_controller, get_user_config,
)

fragment = get_last_fragment()
account_instance = get_account_instance()
messages_controller = get_messages_controller()
connections_manager = get_connections_manager()
send_helper = get_send_messages_helper()
user_cfg = get_user_config()

if user_cfg.getCurrentUser():
    current_user = user_cfg.getCurrentUser()

# Любой из них принимает явный account:
messages_controller = get_messages_controller(1)
```

Исключение: `get_media_controller()` не имеет параметра `account` — `MediaController` единственный глобальный инстанс, а не на аккаунт.

## Мультиаккаунт

exteraGram держит **все** залогиненные аккаунты подключёнными одновременно. Невидимые аккаунты продолжают получать updates, слать запросы и вызывать ваши хуки.

**Проблема:** хелперы `client_utils` по умолчанию используют аккаунт, выбранный в UI, — это глобальное состояние, никак не связанное с аккаунтом, для которого вызван ваш колбэк. Ответ из хука без учёта аккаунта может уйти с другого аккаунта.

### Правило

Каждый хук уже сообщает свой аккаунт:

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
```

Используйте этот `account` для всего, что вы делаете в ответ.

### Таргетинг аккаунта

Все аккаунт-зависимые хелперы принимают опциональный `account`:

```python
from client_utils import get_messages_controller, get_user_config, send_text

get_messages_controller(account)
get_user_config(account)
send_text(peer, "hello", account=account)
```

Когда действий несколько — возьмите клиент: `self.client(...)` возвращает `AccountClient` с теми же методами, привязанный к одному аккаунту. Вне класса плагина то же самое делает `get_client(account)` из `client_utils`.

### Что будет, если забыть

Ничего не «сломается» молча: хелпер без `account` использует выбранный аккаунт, как раньше, но SDK залогирует предупреждение (один раз на хелпер):

```
get_messages_controller() is using account 0 because it is the one selected in the UI,
but it was called while handling account 1. Pass account=1, or use get_client(1),
if you meant that account. Logged once per helper.
```

Скоуп аккаунта распространяется вокруг колбэков хуков, вокруг `run_on_queue`, запланированного из хука, и вокруг `send_request`-колбэков. `get_hook_account()` возвращает его (или `None` вне колбэка), а `self.client()` без аргумента следует ему.

### Осознанное использование выбранного аккаунта

Иногда выбранный аккаунт — именно то, что нужно (например, записать в Saved Messages того, кто смотрит на экран). Скажите это явно — и предупреждение исчезнет:

```python
from client_utils import get_selected_account, send_text

send_text(peer, "note to self", account=get_selected_account())
```

### Делегаты NotificationCenter

`NotificationCenterDelegate.didReceivedNotification` тоже получает `account`, и он **не** скоупится автоматически:

```python
class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args):
        get_client(account).get_messages_controller()
```

### Требование к версии SDK

`account`-параметры, `AccountClient`, `get_client()`, `get_hook_account()` и `get_selected_account()` требуют SDK **1.4.5.0+**. Заявите это:

```python
__sdk_version__ = ">=1.4.5.0"
```

## NotificationCenterDelegate

`NotificationCenterDelegate` — готовый Python-базовый класс для `NotificationCenter.NotificationCenterDelegate`. Слушает клиентские уведомления Telegram.

```python
from java.lang import Object
from java import jarray

from client_utils import NotificationCenterDelegate, get_notification_center
from org.telegram.messenger import NotificationCenter


class MyDelegate(NotificationCenterDelegate):
    def didReceivedNotification(self, id: int, account: int, args: jarray(Object)):
        if id == NotificationCenter.didUpdateConnectionState:
            print("Connection state changed:", args)


delegate = MyDelegate()
notification_center = get_notification_center()
notification_center.addObserver(delegate, NotificationCenter.didUpdateConnectionState)

# Когда больше не нужно:
# notification_center.removeObserver(delegate, NotificationCenter.didUpdateConnectionState)
```

Оборачивать `NotificationCenterDelegate` в `dynamic_proxy(...)` не нужно — достаточно наследования. Если нужен именно сгенерированный Java-подкласс — см. [10. Class Proxy](10-class-proxy.md).

## Bulletins

Для нижних уведомлений — `ui.bulletin.BulletinHelper`:

```python
from ui.bulletin import BulletinHelper

BulletinHelper.show_info("This is an informational message.")
```

Полный список — в [06. Диалоги и уведомления](06-ui-dialogs-and-notifications.md).
