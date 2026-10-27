# 15. Примеры и рецепты из реальных плагинов

Разобранные приёмы из живых плагинов магазина Kangel (775 плагинов) и коллекций сообщества.

> Источники: каталог [Kangel Plugins Store](https://github.com/Kangel-Plugins/Plugins-Store) (775 плагинов, проанализированы все), [extragram_plugins](https://github.com/fan-world-me/extragram_plugins), [awesome-telegram-plugins](https://github.com/makarworld/awesome-telegram-plugins). Каналы для вдохновения: exteraGram Plugins, Dotted (@exteraDev), zwyPlugins (@zwylair).

## Статистика использования API (по 775 плагинам магазина)

| API | Плагинов | Вывод |
|---|---|---|
| `log()` (legacy) | 576 | Большинство до сих пор на legacy-логировании — не значит «правильно» |
| `create_settings` | 555 | Настройки есть у 7 из 10 плагинов |
| `run_on_ui_thread` | 499 | UI-мутации всегда через него |
| `get_last_fragment` | 446 | Стандартный способ достать активный фрагмент |
| `BulletinHelper` | 413 | Уведомления «по умолчанию» |
| `find_class` | 380 | Рефлексия — основа основ |
| `class ...(MethodHook)` | 339 | Xposed-хуки — самая популярная техника |
| `self.hook_method(...)` | 294 | Точечные Java-хуки |
| `dynamic_proxy(...)` | 275 | ⚠️ Широко используется, но задокументирован плохо — только интерфейсы |
| `AlertDialogBuilder` | 268 | Диалоги |
| `on_send_message_hook` | 236 | Команды и приватность |
| `add_menu_item` | 212 | Контекстные меню |
| `get_private_field` | 192 | Доступ к приватным полям Telegram |
| `run_on_queue` | 167 | Фоновая работа |
| `hook_all_methods` | 135 | Хуки всех перегрузок |
| `send_text/send_message` | 123 | Программная отправка |
| `requests` | 112 | HTTP-запросы из плагинов |
| **InMemoryDexClassLoader** | **99** | ⚠️ Продвинутая техника: Java-код из dex-байткода |
| `send_request` (TL) | 93 | Прямые TL-запросы |
| `on_app_event` | 32 | Реакция на START/STOP/PAUSE/RESUME |
| `NotificationCenterDelegate` | 30 | Слушатели событий клиента |
| class proxy (`java_subclass`) | 26 | Генерация Java-подклассов |
| `pre/post_request_hook` | 20 | Перехват запросов |
| новый `self.logger` | 15 | Миграция на новый логгер только началась |
| `edit_message` | 6 | Редактирование используется редко |
| `from elyx import ...` | 2 | Elyx-плагины в магазине пока единичны |

Самые хукуемые TL-методы: `TL_updateNewMessage` (8), `TL_updateNewChannelMessage` (6), `TL_messages_forwardMessages` (4), `TL_updates`/`TL_updatesCombined` (3+3), `TL_updateShortMessage` (3).

## Рецепт: подтверждение отправки команды (paranoia)

Реальный паттерн из `paranoia`: перехват dot-команд, диалог подтверждения, отмена и повторная отправка. Обратите внимание на **приоритет** хука и **защиту от рекурсии**:

```python
def on_plugin_load(self):
    self.add_on_send_message_hook(priority=-1000)  # ранний приоритет

def on_send_message_hook(self, account: int, params: Any) -> HookResult:
    if self._is_resending:
        return HookResult()
    if hasattr(params, "message") and isinstance(params.message, str) and params.message.startswith("."):
        self._pending_send = {
            "peer": getattr(params, "peer", None) or getattr(params, "dialogId", None) or getattr(params, "to_id", None),
            "message": params.message, "replyToMsg": getattr(params, "replyToMsg", None),
            "entities": getattr(params, "entities", None), "media": getattr(params, "media", None),
            "scheduleDate": getattr(params, "scheduleDate", None), "account": account,
        }
        self._show_confirmation_dialog()  # AlertDialogBuilder на UI-потоке
        return HookResult(strategy=HookStrategy.CANCEL)
    return HookResult()
```

При подтверждении — повторная отправка через `send_message(...)`. Поскольку она **снова пройдёт через этот же хук**, флаг сбрасывается отложенно:

```python
self._is_resending = True
try:
    send_message(send_dict)  # peer/message/replyToMsg/entities/media/scheduleDate
finally:
    run_on_ui_thread(lambda: setattr(self, "_is_resending", False), 100)
```

Два урока: (1) набор полей у `SendMessageParams` нестабилен — peer достают цепочкой `getattr` с fallback; (2) при повторной отправке из хука нужна защита от рекурсии с задержкой сброса.

## Рецепт: программная отправка — 4 способа

Из реальных плагинов:

1. **`client_utils.send_message(dict)`** — параметры словарём (использует paranoia, temp_sub).
2. **`client_utils.send_text(dialog_id, text)`** — для простого текста.
3. **Рефлексия приватного `SendMessagesHelper.prepareSendingMedia`** — когда нужен полный контроль над медиа (exereal_forward_edit собирает `SendMessagesHelper$SendingMediaInfo` рефлексией и зовёт приватный метод с точной сигнатурой).
4. **Эмуляция UI** — `edit_text.setText(...)` + `sendButton.performClick()`. Способ «через все хуки»: сообщение проходит полный пайплайн, включая чужие on_send-хуки (smart_sender).

## Рецепт: мьют уведомлений через хук NotificationsController

`mute_person` не даёт уведомлениям появиться вовсе — хук `before_hooked_method` на `NotificationsController.processNewMessages` удаляет заглушённые сообщения из Java-списка **обратным итерированием** (индексы не съезжают):

```python
notify_method = NotificationsController.getClass().getDeclaredMethod(
    "processNewMessages", ArrayList, jbool, jbool, CountDownLatch)  # jbool = Boolean.TYPE
self.hook_notify = self.hook_method(notify_method, ProcessNewMessagesHook(self))

# в before_hooked_method:
for i in range(messages.size() - 1, -1, -1):
    msg = messages.get(i)
    if not self.plugin.store.is_muted(int(msg.getDialogId()), int(msg.getSenderId())):
        continue
    if skip_mentions and bool(msg.messageOwner.mentioned):
        continue
    messages.remove(i)
```

## Рецепт: TL-запросы — подписка на канал, архивация, поиск

`temp_sub` — образец чистой TL-работы через `send_request`:

```python
# Подписка на канал
req = TLRPC.TL_channels_joinChannel()
input_channel = TLRPC.TL_inputChannel()
input_channel.channel_id = channel_id
input_channel.access_hash = access_hash
req.channel = input_channel
send_request(req, RequestCallback(on_join))

# Архивация чата (folder_id=1 — «Архив»)
req = TLRPC.TL_folders_editPeerFolders()
input_peer = TLRPC.TL_inputFolderPeer()
input_peer.peer = mc.getInputPeer(-channel_id)
input_peer.folder_id = 1
req.folder_peers.add(input_peer)
```

Резолв username — `TL_contacts_resolveUsername`, результат кешируется через `mc.putChats()`/`mc.putUsers()`.

`msgsDeleter` — поиск и удаление своих сообщений. Флаги через `req.flags |= N`, удаление **чанками по 100**:

```python
req = TLRPC.TL_messages_search()
req.peer = peer; req.q = ""; req.limit = 100
req.from_id = messages_controller.getInputPeer(user_config.getCurrentUser())
req.flags |= 1                                  # 1 = from_id задан
req.filter = TLRPC.TL_inputMessagesFilterEmpty()
if reply_message_id != 0:
    req.top_msg_id = reply_message_id; req.flags |= 2   # 2 = top_msg_id задан
connections_manager.sendRequest(req, client_utils.RequestCallback(on_response), 2)

# удаление чанками:
for id_chunk in chunked:
    java_list = ArrayList()
    for msg_id in id_chunk:
        java_list.add(jint(msg_id))
    messages_controller.deleteMessages(java_list, None, None, dialog_id,
                                       reply_message_id, True, 0)  # revoke=True
```

## Рецепт: периодические задачи с защитой от дублей

`temp_sub` крутит периодическую проверку через самоперезапускающийся `run_on_queue` с глобальным флагом-стражем (иначе повторный `on_plugin_load` плодит цепочки):

```python
def _run_periodic_check(self, interval_ms):
    if not _check_task_scheduled:
        return
    self._check_and_leave_expired()
    run_on_queue(lambda: self._run_periodic_check(interval_ms), PLUGINS_QUEUE, interval_ms)
```

Альтернатива для точного тайминга — daemon-`threading.Timer` с самоперезапуском; общее состояние между таймером и UI защищайте `threading.Lock`.

## Рецепт: встраивание кнопки в UI предпросмотра отправки

`smart_sender` не использует `on_send_message_hook` вовсе — хук на метод построения меню предпросмотра:

```python
method = MessageSendPreview.getClass().getDeclaredMethod("setItemOptions", ItemOptions)
self.hook_obj = self.hook_method(method, SetItemOptionsHook(self))
# в before_hooked_method: param.args[0] — ItemOptions, param.thisObject — preview
```

А «отправить при прочтении» реализовано поллингом приватных полей:

```python
enter_view = get_private_field(fragment, "chatActivityEnterView")
dialog_id = get_private_field(enter_view, "dialog_id")
account = get_private_field(enter_view, "currentAccount")
dialog = MessagesController.getInstance(account).dialogs_dict.get(dialog_id)
baseline = int(dialog.read_outbox_max_id)   # дальше threading.Timer сравнивает с новым значением
```

## Рецепт: самый сложный UI-хук — модификация меню пересылки

`exereal_forward_edit` показывает потолок техники: `hook_all_constructors(MessagePreviewView, ...)` захватывает `val$previewParams` из anonymous-класса, а `hook_all_methods(ViewGroup, "addView")` находит пункты меню по тексту («Изменить получателя», следом «Не пересылать») и вставляет свою кнопку + клон `GapView` после нужного пункта. Подкласс Java-класса через class proxy:

```python
@java_subclass(PhotoViewer.EmptyPhotoViewerProvider)
class EditForwardPhotoProvider(Base):
    @joverride()
    def sendButtonPressed(self, index, video_edited_info, notify, schedule_date, ttl, force_document):
        ...
```

И вызов приватного метода с точной сигнатурой по типам:

```python
method = helper_class.getDeclaredMethod("prepareSendingMedia", *type_classes)
method.invoke(helper, account_instance, media_list, JLong(dialog_id), None, None, None, None,
              JBoolean(False), JBoolean(group_album), None, JBoolean(not silent_send), ...)
```

## Продвинутая техника: DEX-bridge

99 плагинов магазина грузят Java-байткод из dex. Два подхода из реальных плагинов:

**Полный bridge** (`forwards`): Python-слой только грузит dex и держит мост — вся Xposed-логика живёт в Java:

```python
app_class_loader = ApplicationLoader.applicationContext.getClassLoader()
dex_loader = InMemoryDexClassLoader(ByteBuffer.wrap(dex_bytes), app_class_loader)
cls = dex_loader.loadClass("com.pessdes.forwards.Main")
cls.getDeclaredMethod("main", Utilities.Callback.getClass()).invoke(None, DexLogger())

class DexLogger(dynamic_proxy(Utilities.Callback)):
    def run(self, objs):
        log("[DEX]", *objs)
```

**Образцовая валидация** (`anti_spoiler`): SHA-256 бинарника, проверка magic `b"dex\n"`, лимит размера, контракт моста (`startBridge/updateSettingsBridge/getStateBridge/onUnloadBridge` + статические `BRIDGE_API`/`VERSION_CODE`), выбор лоадера по API-уровню:

```python
if int(Build.VERSION.SDK_INT) >= 26:
    return jclass("dalvik.system.InMemoryDexClassLoader")(ByteBuffer.wrap(dex_bytes), parent)
# иначе — файловый лоадер с атомарной записью:
os.replace(temp_path, dex_path)
return jclass("dalvik.system.DexClassLoader")(dex_path, cache_dir, None, parent)
```

Зачем: тяжёлая логика (перехват отрисовки `ChatMessageCell`, счётчики, обработка спойлеров) в Java работает быстрее и не тормозит Python-поток. Риски: dex — исполняемый код, обязателен hash-контроль; при выгрузке плагина вызывайте `onUnloadBridge` и снимайте хуки.

## Рецепт: синхронный TL-запрос из фонового потока

TL-API асинхронный, но часто нужен синхронный вызов с таймаутом и отменой. Паттерн из `translatica_pro` (нативный перевод) — `threading.Event` + delegate-прокси:

```python
req = TLRPC.TL_messages_translateText()
req.flags |= 2                      # 2 = to_lang задан
req.to_lang = dst
te = TLRPC.TL_textWithEntities()
te.text = masked
te.entities = ArrayList()
req.text.add(te)

done = threading.Event()

class _ReqDelegate(dynamic_proxy(RequestDelegate)):
    def run(self_obj, response, error):
        ...  # сохраняем результат в box
        done.set()

ConnectionsManager.getInstance(int(UserConfig.selectedAccount)).sendRequest(req, _ReqDelegate())
if not done.wait(timeout_sec):      # таймаут 12–18 с
    return None
```

А из `chat_summary` — самый аккуратный вариант: `sendRequest` возвращает `request_id`, его сохраняют в job и на отмене/таймауте вызывают `cancelRequest`:

```python
arrived, box = threading.Event(), []
request_id = send_request(request, callback, account=job.account)
while not arrived.wait(0.1):
    check_cancel(job.cancel)
    if timeout: raise ...
finally:
    if not arrived.is_set() or job.cancel.is_set():
        get_connections_manager(account=job.account).cancelRequest(request_id, True)
```

## Рецепт: надёжные HTTP-запросы

`chat_summary` — эталон hardened-HTTP в плагине:

```python
session = requests.Session()
session.trust_env = False                        # игнорировать .netrc/прокси-ENV
with session.request(method, url, json=payload,
                     timeout=(10, config.timeout),
                     allow_redirects=False, stream=True) as response:
    size = 0
    for chunk in response.iter_content(chunk_size=16384):
        check_cancel(cancel)
        size += len(chunk)
        if size > MAX_HTTP_BYTES:
            raise SummaryError("response_size", ...)
# exponential backoff с уважением Retry-After и джиттером:
delay = max(retry or 0.0, (2 ** attempt) + random.uniform(0.0, 0.5))
```

Простой случай (`currency`) — `requests.get(url, timeout=10)` + кеш в памяти с TTL:

```python
r = requests.get(API_URL.format(base=base, quote=quote), timeout=10)
rate = Decimal(str(r.json()["rate"]))
self.cache[key] = (now, rate)      # dict[(base,quote)] -> (timestamp, value)
```

> ⚠️ `currency` выполняет запрос прямо в `on_send_message_hook` — блокируя отправку. Так делать не стоит: `akres_ip_checker` уводит работу в сетевую очередь `run_on_queue(fn, EXTERNAL_NETWORK_QUEUE)`, а результат показывает через `run_on_ui_thread`.

## Рецепт: параллельные запросы к нескольким API

`akres_ip_checker` опрашивает 5 GeoIP-сервисов одновременно:

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=len(fns)) as ex:
    futures = {ex.submit(fn, ip): i for i, fn in enumerate(fns)}
    for fut in concurrent.futures.as_completed(futures):
        try:
            results_map[futures[fut]] = fut.result()
        except Exception as e:
            results_map[futures[fut]] = f"Ошибка: {e}"
```

И собирает форматирование **вручную** — обратите внимание на UTF-16-длины, обязательные для `MessageEntity`:

```python
e = TLRPC.TL_messageEntityBlockquote()
e.collapsed = True
e.offset = offset
e.length = len(text.encode("utf-16-le")) // 2   # offset/length — в UTF-16-единицах!
send_message({"peer": peer, "message": text, "entities": entities, "searchLinks": False})
```

## Рецепт: встроенный нативный код (.so через ctypes)

Экстремальный уровень из `culprit_detector` и `smart_folder_manager`: arm64/armv7-библиотека хранится base64+zlib, распаковывается в `getCodeCacheDir()` и грузится через ctypes:

```python
raw = zlib.decompress(base64.b64decode(_SO_ARM64))
path = File(context.getCodeCacheDir(), "culprit_native/libculprit.so").getAbsolutePath()
lib = ctypes.CDLL(path)
lib.cd_init.argtypes = [c.c_uint64, c.c_int32, c.c_int32]   # обязательно объявить сигнатуры
lib.cd_init.restype = c.c_int
```

`smart_folder_manager` передаёт массив чатов нативному сортировщику через `ctypes.Structure`:

```python
class ChatData(ctypes.Structure):
    _fields_ = [("dialog_id", ctypes.c_int64), ("last_message_date", ctypes.c_int32), ...]

self.lib.filter_and_sort_chats.argtypes = [
    ctypes.POINTER(ChatData), ctypes.c_int32, SortFilterRules, ctypes.POINTER(ctypes.c_int64)]
```

Проверяйте размер/SHA-256 перед перезаписью файла, а нативную сортировку выполняйте в daemon-потоке с дедупом (`pending_sorts`), копируя Java-список на UI-потоке (`ArrayList(original_list)`) против `ConcurrentModificationException`.

## Рецепт: SQLite с блокировками и миграцией

`smart_folder_manager` / `chat_summary`:

```python
conn = sqlite3.connect(self.db_path, timeout=15.0)
cursor.execute("PRAGMA table_info(folder_configs)")   # проверка схемы перед CREATE/DROP
# + threading.Lock на все записи, версия схемы в настройках
```

## Чек-лист `on_plugin_unload` (эталон — culprit_detector)

Реальный плагин с 30 000 строк останавливает при выгрузке:

1. все `threading.Timer.cancel()`;
2. выставить все `threading.Event` stop-флаги, `thread.join(timeout=2.0)` для reaper-потоков;
3. `self.unhook_method(hook)` для каждого хука, `self.remove_menu_item(...)`;
4. снять monkey-patch'и (`threading.Thread.start = orig`);
5. сбросить открытые диалоги через `run_on_ui_thread`;
6. финальный flush кешей на диск;
7. `SimpleSettingFactory` освободить через `factory.release()` (или вообще не регистрировать);
8. остановить нативную библиотеку (`lib.cd_oom_stop()`), уничтожить свои view (`view.destroy()`), очистить кеши.

## Хуки UI-элементов: приёмы из плагинов кастомизации

**Хук конкретной перегрузки вместо hook_all** (`custom_chats_title`): все перегрузки `setText` внутри сводятся к одной — хукайте её с fallback:

```python
try:
    # Every setText overload funnels into setText(CharSequence, boolean, boolean).
    from java.lang import Boolean
    method = text_view.getDeclaredMethod("setText",
        find_class("java.lang.CharSequence"), Boolean.TYPE, Boolean.TYPE)
    unhook = self.hook_method(method, hook)
except Exception:
    pass
if not unhook:
    self.hook_all(text_view, "setText", hook)     # fallback: все перегрузки
```

**Захват контекста из `param.thisObject`** (`rounder`): before-хук сохраняет `thisObject` (всплывающее окно) и «окружающий» инстанс анонимного класса, after-хук сбрасывает:

```python
def get_surrounding_this(this):
    return get_private_field(this, 'a') or get_private_field(this, 'f$0')
    # 'a' — стандартное имя поля внешнего класса в обфусцированных анонимных классах

class SendButtonHook(MethodHook):
    def before_hooked_method(self, param):
        Plugin.send_button = param.thisObject
        Plugin.editor_instance = get_surrounding_this(param.thisObject)
    def after_hooked_method(self, param):
        Plugin.send_button = None
        Plugin.editor_instance = None
```

Дальше `rounder` читает приватные поля редактора фото: `imagesArrLocals`, `currentIndex` — и модифицирует `ActionBarPopupWindow` до показа через хук `showAtLocation`.

**Что хукуют плагины кастомизации** (7 разобранных): `dispatchDraw`/`dispatchTouchEvent`/`onDraw`/`onLayout` (перерисовка и перехват касаний), `setItemOptions` (меню предпросмотра/прикрепления), `fillItems`/`fillActionModeMenu` (пункты меню), `createView`/`getItemViewType` (построение ячеек), `createClearOrDeleteDialogAlert`, `calcUnreadCounters`, `getMapsProvider` (подмена провайдера карт в `ayu_yandex_maps` — сам провайдер написан на Python через `@java_subclass`), `RenderScript` для размытия (`Combined_Custom_Banner`).

## Общие паттерны (выжимка из 775 плагинов)

1. **Три уровня хуков.** `add_on_send_message_hook(priority=N)` — только исходящие; `hook_method(Class.getDeclaredMethod(...), handler)` — точечный Java-хук с точными типами (`Boolean.TYPE`, `ArrayList`); `hook_all_methods`/`hook_all_constructors` — по имени без сигнатуры. Обязателен `unhook_method` в `on_plugin_unload`.
2. **`isinstance(params.message, str)` — обязательная проверка**: при отправке медиа поле не строка.
3. **Account протаскивается везде**: из хука, из `currentAccount` приватных полей, либо `UserConfig.selectedAccount`. Для `mc.getUser/getChat` — `JLong.valueOf(id)`; положительный `dialog_id` — юзер, отрицательный — чат.
4. **Приватные поля — только через `get_private_field`**, каждый вызов в `try/except`: имена полей меняются между версиями Telegram.
5. **Java-списки мутируются только обратным итерированием** `range(size-1, -1, -1)`.
6. **Настройка → поведение**: `Switch(key=...)` → в хуке `if not self.get_setting("enabled", True): return HookResult()` с ранним выходом. Input возвращает строку — приводите с clamp в try. Структурированные данные (карта мьютов, подписки) — JSON в одной строковой настройке.
7. **Методы для хука ищут рефлексией по списку имён-кандидатов** (`dismiss`/`hide`, `imagePath`/`path`) — устойчивость к расхождению версий.
8. **UI — только через `run_on_ui_thread`; сеть/тяжёлое — `run_on_queue(fn, PLUGINS_QUEUE, delay_ms)`** (для сети лучше `EXTERNAL_NETWORK_QUEUE`); периодика — `threading.Timer` (daemon + cancel в unload) или самоперезапускающиеся цепочки с inflight-флагами; устаревшие результаты отсекаются nonce-счётчиками.
9. **Синхронизация поверх асинхронного TL API**: `threading.Event` + `dynamic_proxy(RequestDelegate)`, результат в `box = []`; на отмене — `cancelRequest(request_id, True)`.
10. **Кеширование трёхуровневое**: память (dict с TTL / `OrderedDict`), файлы (JSON с dirty-флагом и отложенным flush), SQLite (`timeout=15.0` + `threading.Lock` + миграция через `PRAGMA table_info`).
11. **Все offset/length у `MessageEntity` — в UTF-16-единицах**: `len(s.encode("utf-16-le")) // 2`, а не `len(s)`.

## Чего избегать

- не блокируйте UI-поток в хуках — сетевые запросы и файлы только через `run_on_queue` или отдельный поток;
- не забывайте `account` в ответах на хуки — легко отправить сообщение с чужого аккаунта;
- не храните большие данные в настройках плагина — файлы для того и существуют;
- регистрируйте хуки в `on_plugin_load`, а не на уровне модуля — иначе они переживут выгрузку плагина;
- проверяйте `None` из всех рефлексивных вызовов (`find_class`, `get_private_field`) — классы между версиями приложения меняются;
- не грузите dex без hash-проверки — это исполняемый код.
