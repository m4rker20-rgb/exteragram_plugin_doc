# 12. ElyxBuilder и DevServer

CLI-инструмент `elyb`, протокол разработчика и удалённая отладка.

## ElyxBuilder

[ElyxBuilder](https://github.com/shareui/ElyxBuilder) — CLI для создания и упаковки Elyx-проектов: генерация структуры, проверка синтаксиса, сборка source- и compiled-архивов, пересборка при изменениях, списки игнора.

**ElyxBuilder опционален**: рантайму Elyx он не нужен, а корректно свёрстанный ZIP собирается и вручную.

### Установка

Требуется Python 3.10+ (для compiled-сборок `.pyc` — **ровно 3.11**, байткод должен совпадать с рантаймом телефона):

```bash
python -m pip install --upgrade ElyxBuilder
elyb --version
elyb --help
```

### Создание проекта

Интерактивный мастер (спрашивает имя, автора, id, ограничения версий, иконку, форматы refmap/метаданных; Enter — принять сгенерированное):

```bash
mkdir my_plugin && cd my_plugin
elyb new
```

Без вопросов — для скриптов:

```bash
elyb new -g -n "My Plugin" -a myname          # --gen --name --author
elyb new -g -n "My Plugin" -a myname --zipformat elyx   # расширение архива: eaf (по умолчанию) или elyx
```

Генерируемый проект:

```
my_plugin_repository/
├── refmap.yml
└── MyPlugin/
    ├── .elyxbuilder/
    │   └── config.yml
    ├── locales/
    │   ├── strings_en.json
    │   └── strings_ru.json
    ├── meta.yml
    ├── res/
    │   └── .gitkeep
    └── src/
        └── main.py
```

`refmap.yml` остаётся в корне репозитория и архива; все пути ElyxBuilder прописывает в refmap. Перед первой сборкой: замените сгенерированные описания в `locales`, проверьте `id` и версии в `meta.yml`, реализуйте плагин в `src/main.py`. В режиме `-g` id создаётся как `author_PluginName` (до 32 символов), отображаемое имя сохраняется как есть.

### Сборка

Команды запускаются **из директории с `refmap.yml`**.

Source-сборка с проверкой синтаксиса:

```bash
elyb build --ast --verbose --no-folder
# короткая форма:
elyb build -a -v -nf
```

Compiled-сборка (`.py → .pyc`, Python 3.11) с оптимизацией уровня 2:

```bash
elyb build --compile 2 --verbose --no-folder
elyb build -c 2 -v -nf
```

Результат кладётся в `builds/`. `--ast` и `--compile` взаимоисключающие (компиляция уже парсит файлы).

| Опция | Описание |
|---|---|
| `-v`, `--verbose` | Полный лог сборки |
| `-a`, `--ast` | Проверить синтаксис `.py` через AST перед упаковкой |
| `-c [LEVEL]`, `--compile [LEVEL]` | Компилировать `.py → .pyc`; LEVEL 0–2 (по умолчанию 1) |
| `-r`, `--reset` | Очистить кеш компиляции (только с `--compile`) |
| `-nf`, `--no-folder` | Исключить директорию `.elyxbuilder` из архива |
| `--no-assets` | Исключить ресурсы из `optionalAssets` |
| `-ni`, `--no-info` | Не добавлять блок информации ElyxBuilder в meta |
| `-o [FILE...]`, `--obfuscation [FILE...]` | Обфусцировать исходники перед упаковкой (без файлов — весь source) |
| `-p METHOD PASS` | Зашифровать архив, например `-p aes-256 mypassword` |
| `-sv VERSION [APPEND]` | Записать целевую версию; `APPEND=true` добавит `-{version}` к имени архива |
| `-sc PACKAGE [NAME]` | Записать целевой клиент; `NAME` добавит `-{name}` к имени |

> Запускайте `elyb build --help` перед использованием расширенных опций в автоматизации: установленная версия — источник истины по аргументам.

### Watch — пересборка при изменениях

```bash
elyb watch 10 --args "--ast -v -nf"
```

Первый аргумент — интервал опроса в секундах. Последний результат хранится как `builds/latest.eaf` (или с настроенным расширением). В интерактивном экране: `P` — пауза/продолжить, `Q` — выход.

> ⚠️ `elyb watch` **только собирает архив**, не устанавливает его на устройство. Для синхронизации отдельных файлов и перезагрузки установленного плагина используйте DevServer / `elyx_dev_client.py` (ниже).

### Кеш компиляции и игноры

```bash
elyb cached                                  # какие файлы изменились с прошлой compiled-сборки
elyb build --compile --reset                 # чистая компиляция
```

Списки игнора лежат в `.elyxbuilder/config.yml`; обновляйте через CLI:

```bash
elyb add-ignore "MyPlugin/generated/*" --all      # исключить из всех сборок (ignoreAll)
elyb add-ignore "MyPlugin/res/preview.psd" --no-assets  # только при --no-assets (optionalAssets)
elyb add-ignore "MyPlugin/src/compat.py" --compile      # держать файл как source при компиляции
elyb del-ignore 0 --all                           # удалить запись по индексу
```

### Статистика проекта

```bash
elyb stats builds     # сборки
elyb stats lines      # строки кода
elyb stats size
elyb stats files
# --all — весь плагин; --additional DIR... — дополнительные директории
```

### Сборка архива без ElyxBuilder

Любой ZIP-инструмент, сохраняющий пути. Заархивируйте **содержимое** проекта так, чтобы `refmap.yml` был в корне ZIP, затем переименуйте:

```
my_plugin.zip → my_plugin.elyx
```

Перед публикацией проверьте архив: `refmap.yml`, файл метаданных, `main.py`, ресурсы, локали — на верхнем уровне. **Не** отправляйте: `.git/`, `.elyx/cache/`, `builds/`, `__pycache__/`, десктопные виртуальные окружения, секреты и логи.

## DevServer (сервер разработки)

SDK включает небольшой TCP-сервер для десктопных инструментов: инспекция и перезагрузка обычных Python-плагинов; Elyx расширяет его мультифайловой синхронизацией.

Дефолтная точка: `127.0.0.1:42690`. Сервер слушает loopback телефона — подключайтесь через ADB:

```bash
adb devices
adb forward tcp:42690 tcp:42690
```

> ⚠️ **Только для разработки.** Протокол умеет устанавливать, переписывать, включать/выключать, перезагружать и удалять код плагинов. Включайте сервер только пока разрабатываете.

Если порт занят другим процессом — уберите и пересоздайте forward:

```bash
adb forward --remove tcp:42690
adb forward tcp:42690 tcp:42690
```

### Формат сообщений

Соединение несёт последовательные UTF-8 JSON-объекты:

```json
{ "@": "ping", "#": "request-1" }
```

- `@` — имя команды;
- `#` — клиентский id запроса;
- остальные поля зависят от команды. Ответ повторяет id запроса:

```json
{ "#": "request-1", "pong": true }
```

### Базовые команды

| Команда | Поля | Назначение |
|---|---|---|
| `ping` | — | Проверка соединения |
| `get_plugins` | — | Список загруженных плагинов и состояний |
| `enable_plugin` / `disable_plugin` | `plugin_id` | Включить/выключить |
| `reload_plugin` | `plugin_id` | Перезагрузить однофайловый плагин |
| `write_plugin` | `plugin_id`, `content` | Установить/обновить Python-исходник (полный текст строкой JSON) |
| `remove_plugin` | `plugin_id` | Удалить плагин |
| `start_debugger` | `host`, `port`, `platform` | Запустить удалённый отладчик |
| `stop_debugger` | `platform` | Остановить отладчик |

`write_plugin.content` пишется во временный файл и проходит обычный загрузчик плагинов.

### Elyx-команды

| Действие | Назначение |
|---|---|
| `elyx_ping` | Проверить, что Elyx-патч сервера активен |
| `get_elyx_plugins` | Метаданные загруженных Elyx-плагинов |
| `elyx_compare_folder` | Сравнить клиентские хеши с установленными файлами |
| `elyx_changes` | Применить изменения файлов и перезагрузить плагин |

Elyx-пейлоады — сжатый (zlib) base64 JSON в поле `data`. Запись изменений:

```json
{
  "plugin_id": "elyx_hello",
  "changes": [
    { "#": "modified", "path": "main.py", "is_directory": false, "content": "<base64 file bytes>" }
  ]
}
```

Типы событий: `created`, `modified`, `deleted`, `moved` (у moved — `path` и `dest_path`). Пути относительны корня плагина. После применения батча Elyx выгружает плагин, чистит его модули, загружает заново и отвечает `success` + список `errors`.

### Удалённая отладка

```json
{ "@": "start_debugger", "#": "debug-1", "host": "127.0.0.1", "port": 5678, "platform": "vscode" }
```

Платформы: `vscode`, `pycharm`. Порт отладчика (`5678`) — отдельный от `42690`; настройте ADB forward/reverse под направление, которое требует отладчик. Отладка также требует, чтобы соответствующий Python-debugger-пакет был доступен в рантайме приложения (debugpy предустановлен).

VS Code `launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to exteraGram",
            "type": "debugpy",
            "request": "attach",
            "connect": { "host": "127.0.0.1", "port": 5678 },
            "pathMappings": [
                {
                    "localRoot": "${workspaceFolder}",
                    "remoteRoot": "/data/user/0/com.exteragram.messenger/files/plugins"
                }
            ]
        }
    ]
}
```

## Цикл разработки Elyx (live reload)

Рабочий цикл: правите проект на ПК → синхронизируете изменённые файлы в установленный плагин → Elyx выгружает и загружает плагин → смотрите ошибки → правите дальше.

### Предпосылки

- свежий exteraGram с поддержкой плагинов;
- текущий SDK доступен IDE (или `pip install exteragram-utils`);
- Python 3.11 для байткод-совместимых сборок;
- ADB для USB-устройства/эмулятора;
- **Elyx-архив, установленный на устройстве один раз**.

### Первоначальная установка

Протокол live reload обновляет Elyx-плагин, уже известный движку. Перед запуском вотчера:

1. соберите `.elyx`/`.eaf`-архив;
2. установите его через exteraGram;
3. включите плагин хотя бы раз;
4. убедитесь, что `id` в локальном `metainfo` совпадает с установленным.

### Клиент синхронизации

Официальный клиент (`elyx_dev_client.py`) **не входит в SDK-репозиторий** — держите его версию рядом с проектом или в общем репо инструментов. Клиенту нужны: host/port (обычно `127.0.0.1:42690` после forward), локальный корень проекта, `id` плагина, игноры (`.git`, `.elyx`, `builds`, `__pycache__`, файлы IDE). Поведение — инкрементальное: сравнение хешей → отправка created/modified/moved/deleted → ожидание перезагрузки от движка.

В community-экосистеме есть и альтернативные воркфлоу, например `catalib watch --deploy` для однофайловых плагинов.

## Правила reload-safe кода

Elyx чистит локальные Python-модули, Java-хуки и пункты меню. Ответственность за созданные вами ресурсы — на вас:

```python
class ReloadSafePlugin(BasePlugin):
    def on_plugin_load(self):
        self.closed = False
        self.worker = create_worker()
        self.worker.start()

    def on_plugin_unload(self):
        self.closed = True
        if self.worker:
            self.worker.stop()
            self.worker = None
```

Не запускайте постоянную работу на import-времени модуля: инициализация — в `on_plugin_load`, очистка — в `on_plugin_unload`.

## Чеклист релиза

- [ ] установка из чистого архива (не только через live reload);
- [ ] тест дефолтной локали и английского fallback;
- [ ] тест без сети, если есть зависимости;
- [ ] выключение/включение плагина;
- [ ] обновление с предыдущей публичной версии;
- [ ] все объявленные пути ресурсов и локализаций проверены;
- [ ] в корне архива нет лишней директории-обёртки;
- [ ] `id`, ограничения версий и ссылки зависимостей подтверждены.
