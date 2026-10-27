# 04. Хуки событий

Перехват Telegram-запросов, ответов, обновлений и исходящих сообщений.

## Типы хуков

Хуки позволяют плагину перехватывать:

- **запросы к API** (до и после отправки);
- **ответы** сервера;
- **обновления** (updates) — одиночные и контейнеры;
- **параметры исходящих сообщений**.

Регистрация:

```python
self.add_hook(name, match_substring=False, priority=0)
self.add_on_send_message_hook(priority=0)
```

Имена хуков — это имена TL-методов/обновлений, например `TL_messages_setTyping`, `TL_updateNewMessage`.

```python
class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_hook("TL_messages_setTyping")
        self.add_hook("TL_messages_setEncryptedTyping")
        self.add_hook("TL_account_updateStatus")
        self.add_on_send_message_hook()
```

## Обработчики

### pre_request_hook — до отправки запроса

```python
def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
    ...
```

### post_request_hook — после ответа

```python
def post_request_hook(self, request_name: str, account: int, response: Any, error: Any) -> HookResult:
    ...
```

### on_update_hook / on_updates_hook — обновления

```python
def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
    result = HookResult()
    if update_name == "TL_updateNewMessage":
        self.logger.info(f"Intercepted update: {update_name}")
    return result


def on_updates_hook(self, container_name: str, account: int, updates: Any) -> HookResult:
    result = HookResult()
    if container_name == "TL_updates":
        self.logger.info(f"Received updates container: {container_name}")
    return result
```

### on_send_message_hook — исходящие сообщения

```python
def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    result = HookResult()
    if hasattr(params, "message") and isinstance(params.message, str):
        if params.message.startswith(".hello"):
            params.message = params.message.replace(".hello", "Hello from plugin")
            result.strategy = HookStrategy.MODIFY
            result.params = params
    return result
```

## Полный пример: Ghost Mode

```python
from typing import Any, List

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Switch


TYPING_REQUESTS = ["TL_messages_setTyping", "TL_messages_setEncryptedTyping"]


class GhostModePlugin(BasePlugin):
    def on_plugin_load(self):
        for req_name in TYPING_REQUESTS:
            self.add_hook(req_name)
        self.add_hook("TL_account_updateStatus")

    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        if request_name in TYPING_REQUESTS and self.get_setting("dont_send_typing", True):
            self.logger.info(f"Blocking request: {request_name}")
            return HookResult(strategy=HookStrategy.CANCEL)

        if request_name == "TL_account_updateStatus" and self.get_setting("force_offline", True):
            request.offline = True
            return HookResult(strategy=HookStrategy.MODIFY, request=request)

        return HookResult()

    def post_request_hook(self, request_name: str, account: int, response: Any, error: Any) -> HookResult:
        if request_name == "TL_messages_sendMessage" and not error:
            self.logger.info("Successfully sent a message!")
        return HookResult()

    def create_settings(self) -> List[Any]:
        return [
            Switch(key="dont_send_typing", text="Don't send typing status", default=True),
            Switch(key="force_offline", text="Always appear offline", default=True),
        ]
```

## Стратегии `HookStrategy`

Результат хука возвращается как `HookResult` со стратегией:

| Стратегия | Поведение |
|---|---|
| `HookStrategy.DEFAULT` | Ничего не менять |
| `HookStrategy.CANCEL` | Остановить операцию |
| `HookStrategy.MODIFY` | Вернуть изменённый объект |
| `HookStrategy.MODIFY_FINAL` | Вернуть изменённый объект и остановить обработку другими плагинами |

В зависимости от типа хука изменённый объект кладётся в поле `HookResult`:

- `HookResult.request` — запрос;
- `HookResult.response` — ответ;
- `HookResult.update` — одиночное обновление;
- `HookResult.updates` — контейнер обновлений;
- `HookResult.params` — параметры исходящего сообщения.

Для исходящих сообщений:

- `HookResult()` — «оставить как есть»;
- `HookResult(strategy=HookStrategy.MODIFY, params=params)` — «использовать изменённые параметры»;
- `HookResult(strategy=HookStrategy.CANCEL)` — «отменить отправку полностью».

## Мультиаккаунт в хуках

Каждый хук получает параметр `account`, и это **часто не тот аккаунт, который выбран в UI** — фоновые аккаунты остаются на связи и продолжают вызывать хуки.

- Изменение перехваченного объекта (как в примерах выше) — безопасно без лишних действий.
- Как только вы **отправляете, запрашиваете или читаете что-то в ответ** — используйте `self.client(account)` или явный `account=`.

```python
def on_update_hook(self, update_name: str, account: int, update) -> HookResult:
    if update_name != "TL_updateNewMessage":
        return HookResult()

    client = self.client(account)  # клиент, привязанный к account хука
    me = client.get_user_config().getCurrentUser()
    client.send_text(update.message.dialog_id, f"replying as {me.first_name}")
    return HookResult()
```

Подробно — в [05. Клиентские утилиты](05-client-utils.md#мультиаккаунт) и разделе про мультиаккаунт там же.

## Типичные ошибки новичков

- забыть `self.add_on_send_message_hook()` в `on_plugin_load` — реализация `on_send_message_hook` сама по себе ничего не делает;
- строить метаданные динамически вместо констант;
- считать, что `params.message` всегда существует — некоторые отправки без текста (медиа);
- тяжёлая сетевая работа или файловый I/O прямо в хуке — хук работает в горячем пути; выносите работу в `run_on_queue(...)` (см. [05](05-client-utils.md)).
