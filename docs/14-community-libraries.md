# 14. Библиотеки и инструменты сообщества

AltyLib, PACLibs, zwylib, mandre_lib и другие надстройки над SDK + механика «библиотека-плагин».

## Как устроены библиотеки-плагины (изнутри)

Движок exteraGram загружает каждый `.plugin` как Python-модуль с именем `__id__` и кладёт в `sys.modules`. Поэтому **библиотека — это обычный плагин** (класс `BasePlugin` + метаданные), которого другие плагины импортируют по id:

```python
# Потребитель (zwyAdminTools) — эталон graceful-деградации:
try:
    import zwylib
    if not zwylib.is_zwylib_version_sufficient(__name__, REQUIRED_ZWYLIB_VERSION, notify=False):
        raise Exception("zwylib too old")
    from zwylib import UI, SettingEntry, SettingsManager, Requests, PluginUtils
except Exception:
    ...  # заглушки + bulletin "ZwyLib required"
```

Простой вариант (`mandre_encrypt`):

```python
try:
    from mandre_lib import Mandre, MandreInstall
except ImportError:
    Mandre = None
```

Проверить, установлен ли плагин, можно через `sys.modules` (так делает `.help` в `dont65_lib`): `p_id in sys.modules` — плагин загружен в этой сессии.

Путь к файлу плагина и внутренности контроллера — через `PluginsController`:

```python
from com.exteragram.messenger.plugins import PluginsController

controller = PluginsController.getInstance()
engines = controller.engines                    # {"python": PythonPluginsEngine, ...}
path = controller.getPluginPath(plugin_id)      # путь к .plugin файлу
plugin = controller.plugins.get(plugin_id)      # getVersion()/getName()/isEnabled()
```

### Сравнение подходов реальных библиотек

| Библиотека | Подход | Ключевые API |
|---|---|---|
| **zwylib** (@zwylair) | «Инфраструктура + самовосстановление»: хукит сам движок плагинов, перезагружает зависимые плагины | `UI`, `SettingsManager`, `Requests`, `DialogPicker`, `command_manager`, `async_manager`, `CacheFile`, `add_autoupdater_task()` |
| **mandre_lib** | «Жирный фасад»: один импорт `Mandre` = 25+ подсистем | `Mandre.UI/TTS/STT/Data/Text/Auth/Share/Color/Hidden/Device/Async/Reflect/Sensors/Graphics/Command/Settings/Animation/Messages/Dex/Send` |
| **excomponents_lib** | Декларативный UI-кит, ленивые импорты — все Android-классы резолвятся через `find_class()` внутри функций, что даёт устойчивость к версиям | `EXText/EXColumn/EXCard/EXScroll/EXBanner/EXMetric/EXActionRow/EXSection/EXScreen` |
| **moonlib** | Хелперы Java-взаимодействия, грузится рано (`__priority__ = 1`) | `HookRouter`, `TaskManager`, `JavaExceptions`, `LoadClass`, `AdvancedBulletinHelper` |
| **cactuslib_mini** | Минимализм: две текстовые команды + доступ к магазину плагинов | `PluginsController`-хелперы, отправка `.plugin`-файла в чат |
| **dont65_lib** | «Библиотека-сервис»: статические утилиты, привязанные к хранилищу настроек | `Dont65Utils`, агрегированный `.help` по `sys.modules` |

### Паттерн zwylib: библиотека с самовосстановлением

Зависимые плагины падают, если библиотека ещё не загружена. `zwylib` после старта сканирует исходники всех плагинов на упоминание себя и перезагружает упавшие:

```python
def _does_plugin_require_zwylib(plugin_id):
    source = open(PluginUtils.get_plugin_path(plugin_id), encoding="utf-8", errors="ignore").read().lower()
    return "import zwylib" in source or "from zwylib import" in source

# в on_plugin_load (не мешая старту приложения — отсечка 15 сек):
for plugin_id, plugin in plugins.items():
    err = plugin.getError()
    if (err and "zwylib" in err.getMessage().lower()) or _does_plugin_require_zwylib(plugin_id):
        run_on_queue(lambda pid=plugin_id: PluginUtils.reload_plugin(pid), delay=2000)
```

Команды зависимых плагинов при этом «переживают» перезагрузку: `command_manager.mark_as_not_temporal()` + `load_pending_commands()` + свой `on_send_message_hook(priority=999)`, а хуки на `PythonPluginsEngine.unloadPlugin` и `PluginsController.setPluginEnabled` дают библиотеке знание о жизненном цикле любых плагинов.

### Регистрация команд в библиотеках

Вся командная работа зависимых плагинов идёт через перехват исходящих сообщений самой библиотекой:

```python
# потребитель:
dp = zwylib.command_manager.get_dispatcher()   # plugin_id определяется из стека

@dp.register_command("ping", aliases=["p"])
def ping(param, account): ...                  # сигнатура (param, account) — account обязателен
```

## PACLibs

**[PACLibs](https://github.com/PacHub228/PACLibs)** — библиотека-инструмент (не устанавливаемый плагин): хуки Java-методов одной функцией, роутер dot-команд, удаление сообщений, локальная верификация, снятие `FLAG_SECURE`, JSON-хранилище, буллетины.

### Подключение

**Способ 1 (безопасный, рекомендуемый): файлом рядом.** Скачайте `paclibs.py`, положите рядом с плагином:

```python
from paclibs import PAC
```

**Способ 2: загрузка по ссылке с проверкой целостности.** URL привязан к конкретному коммиту + SHA-256 проверка перед `exec`. Подробный код лоадера — в README репозитория.

### Основные возможности

| API | Назначение |
|---|---|
| `PAC.hook(plugin, "класс", "метод", before=..., after=...)` | Xposed-хук без своего класса `MethodHook`; возвращает хендлы |
| `PAC.info/success/error(text)` | Буллетины |
| `PAC.dialog_id_of(msg)` / `PAC.dialog_id_from_peer(peer)` | Вычисление `dialog_id` из MessageObject/TLRPC.Peer |
| `PAC.delete_message(msg_id, dialog_id, revoke=False)` | Удаление сообщения (`revoke=True` — у всех) |
| `PAC.force_verified_user/chat(id)` | Локальная синяя галочка |
| `PAC.allow_screenshots(plugin)` | Снятие `FLAG_SECURE` |
| `PAC.Commands` | Роутер команд: `@cmds.on(".ping")` + `cmds.process(params)` |
| `PAC.load_set/save_set(plugin, key)` | Множества, сериализованные в настройки |

Каждый хелпер обёрнут в `try/except` — ошибка внутри не роняет плагин.

## AltyLib

**[AltyLib](https://github.com/Altuskhins/AltyLib)** — расширенная библиотека: DSL настроек, типизированные заглушки Telegram-контроллеров, менеджер безопасных хуков, форматирование, кеширование, UI-компоненты, CLI-генератор каркаса.

```python
import altylib as alty

settings = alty.create_settings_registry(__name__)

@settings.switch(default=True, summary="Отвечать на команду .hello")
def enabled(value: bool):
    alty.log(f"Hello command toggled to {value}")

class Hello(BasePlugin):
    def on_plugin_load(self):
        alty.register_command("hello", self.cmd_hello, "Поздороваться")

    def on_send_message_hook(self, account, params):
        return alty.handle_outgoing_command(self, account, params) or HookResult()
```

| Категория | API |
|---|---|
| UI | `bulletin_info/error/success`, `show_spinner/hide_spinner`, `quick_list_pick`, `show_bottom_sheet` |
| Команды | `register_command`, `handle_outgoing_command` |
| Event Bus / RPC | `events_subscribe/publish`, `rpc_register/call` — общение между плагинами |
| Данные | `KVStore`, `JsonCacheFile`, `TTLCache`, `shared_set/get/delete` |
| Telegram | `resolve_peer()`, `pick_dialog()`, `send_text()`, `download_media()` |
| Rate-limit | `debounce()`, `throttle()`, `RateLimiter` (anti-flood для TL-запросов) |
| Диагностика | `install_crash_reporter()`, `enable_hot_reload()` |

## Продвинутые техники из экосистемы (с оценкой рисков)

### Загрузка DEX (99 плагинов магазина используют dex-лоадеры)

Полный цикл на эталонном примере `account_hider` (весь код плагина — 46 строк):

```python
DEX_B64 = """ZGV4CjAzOAA0fxOf..."""  # .dex, встроенный в файл плагина

def on_plugin_load(self):
    dex_bytes = base64.b64decode(DEX_B64)
    buffer = ByteBuffer.wrap(dex_bytes)
    parent_cl = ApplicationLoader.applicationContext.getClassLoader()
    self.dex_loader = InMemoryDexClassLoader(buffer, parent_cl)
    self.dex_class = self.dex_loader.loadClass("dev.rooni.accounthider.AccountHider")
    self.dex_instance = self.dex_class.newInstance()
    self.dex_class.getMethod("start").invoke(self.dex_instance)

def on_plugin_unload(self):                 # симметричная выгрузка
    self.dex_class.getMethod("stop").invoke(self.dex_instance)
    self.dex_class = self.dex_loader = self.dex_instance = None
```

Варианты доставки dex из реальных плагинов:

| Вариант | Пример | Особенности |
|---|---|---|
| Inline base64 в исходнике | `account_hider` | Нет сети (плюс к безопасности), но бинарный код непрозрачен для аудита |
| base64 + zlib в комментариях между маркерами `# __DEX_BEGIN__` / `# __DEX_END__`, читается через `__file__` | `AtmosFX` | Компактно; лоадер кешируется в `sys._atmosfx_dex_cache`, чтобы пережить перезагрузку плагина (классы в Java нельзя выгрузить) |
| Скачивание по URL в кеш с проверкой `Content-Length` и атомарной заменой `.tmp` + `os.replace` | `TheLooker` | Обновляемость; ⚠️ RCE-вектор без проверки подписи/хеша |

Проверка версии моста (контракт Python ↔ Java) из `anti_spoiler`:

```python
bridge_api = self._static_int(dex_class, "BRIDGE_API")
if bridge_api != EXPECTED_BRIDGE_API:
    raise RuntimeError(f"bridge API {bridge_api}, ожидался {EXPECTED_BRIDGE_API}")
```

### Обход Hidden API restrictions

`hiddenApiBypass` встраивает dex с **LSPosed HiddenApiBypass** (`org.lsposed.hiddenapibypass.*`) и оборачивает в Python-фасад с кешированием `Method`-объектов и флагом `inited` в Java-поле (защита от повторной инициализации при перезагрузке плагина — DEX-класс остаётся в classloader'е приложения):

```python
dex_loader = InMemoryDexClassLoader(ByteBuffer.wrap(base64.b64decode(cleaned)),
                                    ApplicationLoader.applicationContext.getClassLoader())
clazz = dex_loader.loadClass("ni.shikatu.hiddenapi.HiddenApiWrapper")
clazz.getMethod("init").invoke(None)

HiddenApiBypass.getDeclaredMethods(clazz)      # приватные методы без ограничений
HiddenApiBypass.invoke(clazz, thiz, "methodName", args)
```

Ту же обёртку (`ni.shikatu.hiddenapi.HiddenApiWrapper`) использует `mandre_lib.MandreHidden`, но dex скачивает с GitHub и кеширует в `cacheDir`.

### Runtime-pip в mandre_lib

`MandrePip` скачивает wheels pip/setuptools, распаковывает в `files/mandre_pip/site-packages`, добавляет в `sys.path`, вызывает `pip._internal.cli.main` с перехватом stdout/stderr. ⚠️ **Высокий риск**: установка произвольного кода в процесс мессенджера; работает только для pure-Python wheels; ломается при обновлениях Chaquopy.

### Сводка рисков продвинутых техник

| Техника | Риск |
|---|---|
| Dex, скачиваемый по URL | **Максимальный** — автор плагина (или компрометация хостинга) исполняет произвольный Java-код на устройстве. Требуйте hash-проверку |
| Runtime-pip | Высокий — произвольный код в процессе; чистые wheels только |
| Hidden API bypass (Unsafe, мета-рефлексия) | Высокий — приватные API меняются между версиями Android; возможны краши процесса; API 26+ для in-memory dex |
| Встроенный (inline) dex | Средний — нет сети, но код непрозрачен; классы нельзя выгрузить из памяти |
| Хуки на движок плагинов (`PythonPluginsEngine`, `PluginsController`) | Средний — хрупко к обновлениям exteraGram |
| Кеш dex-лоадера в `sys` | Низкий — классы остаются в памяти до смерти процесса |

## Инструменты разработки

| Инструмент | Что делает | Где |
|---|---|---|
| `exteragram-utils` | Типизированные заглушки SDK для IDE: `pip install exteragram-utils` | PyPI |
| [ElyxBuilder](https://github.com/shareui/ElyxBuilder) | Генерация и сборка Elyx-проектов (`elyb`) — см. [12](12-elyxbuilder-and-dev-server.md) | GitHub / PyPI |
| `catalib` | Деплой плагинов: `catalib watch --deploy` | [документация](https://raito-kyokai.gitbook.io/catalib/deploi/workflow.md) |
| [Pluggy Bot](https://t.me/pluggy_robot) | Статический анализ безопасности плагинов | Telegram |

## Как выбрать библиотеку

- **Минимальный плагин** — чистый SDK, ничего подключать не нужно.
- **Много dot-команд, UI-скриптов** — PACLibs (лёгкая, можно положить файлом рядом).
- **Крупный проект с настройками, планировщиком, событиями** — AltyLib.
- **Команды и UI-кит от @zwylair** — zwylib (учтите: зависимость от стороннего плагина).
- **Максимум подсистем одним импортом** — mandre_lib (осторожно с runtime-pip и скачиваемым dex).
- **Сборка/деплой** — ElyxBuilder (Elyx-проекты), catalib (деплой однофайловых).

При публикации плагина, зависящего от сторонней библиотеки, явно указывайте это в описании и проверяйте версию библиотеки при импорте (паттерн `is_zwylib_version_sufficient`).
