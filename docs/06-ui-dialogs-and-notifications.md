# 06. UI: диалоги, уведомления, Android-хелперы

AlertDialogBuilder, BulletinHelper и `android_utils`.

## Alert Dialog Builder (`ui.alert`)

`AlertDialogBuilder` — Pythonic-обёртка над `org.telegram.ui.ActionBar.AlertDialog.Builder`.

### Базовое использование

```python
from ui.alert import AlertDialogBuilder
from client_utils import get_last_fragment
from android_utils import log

current_fragment = get_last_fragment()
if not current_fragment:
    log("Cannot show dialog, no current fragment.")
    # return / handle error

activity = current_fragment.getParentActivity()
if not activity:
    log("Cannot show dialog, no parent activity.")

builder = AlertDialogBuilder(activity)  # по умолчанию ALERT_TYPE_MESSAGE
builder.set_title("My Plugin Alert")
builder.set_message("This is an important message from the plugin.")

def on_positive_click(bld: AlertDialogBuilder, which: int):
    log("Positive button clicked!")
    bld.dismiss()

def on_negative_click(bld: AlertDialogBuilder, which: int):
    log("Negative button clicked!")
    bld.dismiss()

builder.set_positive_button("OK", on_positive_click)
builder.set_negative_button("Cancel", on_negative_click)

# create() необязателен, но полезен, если нужно настроить флаги до show()
builder.create()
builder.set_cancelable(True)
builder.show()
```

### Типы диалогов

Задаются параметром `progress_style` конструктора:

- `AlertDialogBuilder.ALERT_TYPE_MESSAGE` (по умолчанию) — обычное сообщение;
- `AlertDialogBuilder.ALERT_TYPE_LOADING` — детерминированный горизонтальный прогресс (`set_progress(value)`);
- `AlertDialogBuilder.ALERT_TYPE_SPINNER` — неопределённый спиннер (loading-состояния).

```python
loading_builder = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_SPINNER)
loading_builder.set_title("Loading Data...")
loading_builder.set_message("Please wait while data is being fetched.")
loading_builder.show()
loading_builder.set_cancelable(False)  # лучше после create() или show()
# ...когда загрузка закончилась:
# loading_builder.dismiss()
```

### Ключевые методы

**Инициализация:** `AlertDialogBuilder(context: Context, progress_style: int = ALERT_TYPE_MESSAGE, resources_provider: Optional[Theme.ResourcesProvider] = None)`

**Содержимое:**

- `set_title(title)` — заголовок;
- `set_message(message)` — основной текст;
- `set_message_text_view_clickable(clickable)` — сделать текст кликабельным (ссылки);
- `set_view(view, height=-2)` — кастомный `View` как содержимое;
- `set_items(items: List[str], listener=None, icons=None)` — список пунктов; listener получает `(builder, индекс_пункта)`.

**Кнопки:**

- `set_positive_button(text, listener=None)` / `set_negative_button(...)` / `set_neutral_button(...)`;
- listener получает `(AlertDialogBuilder, идентификатор_кнопки)` — константы `BUTTON_POSITIVE` и т.д.;
- `make_button_red(button_type)` — красный текст кнопки (`Theme.key_text_RedBold`).

**Слушатели:**

- `set_on_back_button_listener(listener)` — нажатие «назад» при открытом диалоге;
- `set_on_dismiss_listener(listener)` — при закрытии по любой причине;
- `set_on_cancel_listener(listener)` — при отмене (назад/тап вне).

**Внешний вид и поведение:**

- `set_top_image(res_id, background_color)`, `set_top_drawable(drawable, background_color)`;
- `set_top_animation(res_id, size, auto_repeat, background_color, layer_colors=None)`;
- `set_dim_enabled(enabled)`, `set_dialog_button_color_key(theme_key)`;
- `set_blurred_background(blur, blur_behind_if_possible=True)`;
- `set_cancelable(cancelable)`, `set_canceled_on_touch_outside(cancel)` — вызывать после `create()`/`show()`.

**Жизненный цикл:**

- `create() -> AlertDialogBuilder` — создать без показа;
- `show() -> AlertDialogBuilder` — создать (если нужно) и показать;
- `dismiss()` — закрыть, если показан;
- `get_dialog() -> Optional[AlertDialog]` — Java-инстанс или `None` после освобождения;
- `get_button(button_type) -> Optional[View]` — кнопка для стилизации (после `create()`/`show()`).

**Прогресс:** `set_progress(progress: int)` — для `ALERT_TYPE_LOADING` (0–100).

### Пример: диалог со списком

```python
from ui.alert import AlertDialogBuilder


def on_item_click(bld: AlertDialogBuilder, which: int):
    items_list = ["Option A", "Option B", "Option C"]
    log(f"Item '{items_list[which]}' (index {which}) selected.")
    bld.dismiss()

item_builder = AlertDialogBuilder(activity)
item_builder.set_title("Choose an Option")
item_builder.set_items(["Option A", "Option B", "Option C"], on_item_click)
item_builder.set_negative_button("Cancel", lambda b, w: b.dismiss())
item_builder.show()
```

### Важные замечания

- **Context:** всегда передавайте валидный `Context` (обычно `Activity`) — `get_last_fragment().getParentActivity()`.
- **Listeners:** первый аргумент колбэка — Python-инстанс `AlertDialogBuilder`, из него можно вызвать `bld.dismiss()`.
- **Потоки:** манипуляции с диалогом — на UI-потоке. Из фона используйте `run_on_ui_thread`.
- **Один диалог на builder:** после закрытия builder держит диалог только слабой ссылкой; повторно показать закрытый диалог нельзя — создавайте новый `AlertDialogBuilder` каждый раз.
- Ошибки в ваших колбэках логируются и не роняют приложение.

## Bulletin Helper (`ui.bulletin`)

`BulletinHelper` — статические методы для показа Telegram-«бюллетеней»: маленьких ненавязчивых уведомлений внизу экрана, исчезающих автоматически.

Если фрагмент не передан, хелпер перебирает fallback-варианты: переданный фрагмент → текущий безопасный фрагмент `LaunchActivity` → видимый bulletin bottom sheet → глобальная фабрика.

```python
from ui.bulletin import BulletinHelper
from client_utils import get_last_fragment
from org.telegram.messenger import R as R_tg  # для Lottie-анимаций Telegram

current_fragment = get_last_fragment()  # необязательно

BulletinHelper.show_info("This is some information.", current_fragment)
BulletinHelper.show_error("An error occurred processing your request.", current_fragment)
BulletinHelper.show_success("Action completed successfully!", current_fragment)
```

> ✅ Все `show_...` методы сами гарантируют выполнение на UI-потоке — оборачивать в `run_on_ui_thread` не нужно.

### Стандартные

- `show_info(message, fragment=None)` — иконка info (`R.raw.info`);
- `show_error(message, fragment=None)` — иконка ошибки;
- `show_success(message, fragment=None)` — иконка успеха.

### Кастомные простые

- `show_simple(text, icon_res_id, fragment=None)` — одна строка + Lottie-иконка;
- `show_two_line(title, subtitle, icon_res_id, fragment=None)` — две строки + иконка.

```python
BulletinHelper.show_simple("Processing...", R_tg.raw.timer, current_fragment)
BulletinHelper.show_two_line("Download Complete", "File saved to gallery.", R_tg.raw.ic_download_done, current_fragment)
```

### С кнопками

- `show_with_button(text, icon_res_id, button_text, on_click=None, fragment=None, duration=DURATION_PROLONG)`:

```python
def open_settings_action():
    print("Settings button clicked!")

BulletinHelper.show_with_button(
    "Plugin settings updated.", R_tg.raw.info, "Configure", open_settings_action, current_fragment,
)
```

- `show_undo(text, on_undo, on_action=None, subtitle=None, fragment=None)` — «Undo»-уведомление; `on_action` вызывается после задержки, если Undo не нажали:

```python
def perform_delete():
    print("Item permanently deleted.")

def undo_delete():
    print("Delete operation undone.")

BulletinHelper.show_undo("Item moved to trash.", on_undo=undo_delete, on_action=perform_delete, fragment=current_fragment)
```

### Контекстные (предопределённые)

- `show_copied_to_clipboard(message=None, fragment=None)`;
- `show_link_copied(is_private_link_info=False, fragment=None)`;
- `show_file_saved_to_gallery(is_video=False, amount=1, fragment=None)`;
- `show_file_saved_to_downloads(file_type_enum_name="UNKNOWN", amount=1, fragment=None)` — имя из `BulletinFactory.FileType`, например `"MUSIC"`, `"PHOTO_TO_DOWNLOADS"`, `"GIF"`.

### Длительности

- `BulletinHelper.DURATION_SHORT` — 1500 мс;
- `BulletinHelper.DURATION_LONG` — 2750 мс;
- `BulletinHelper.DURATION_PROLONG` — 5000 мс.

### Поиск Lottie-анимаций

Иконки бюллетеней — Lottie-анимации из raw-ресурсов. Смотрите `TMessagesProj/src/main/res/raw/` в исходниках Telegram. В Python — `org.telegram.messenger.R.raw.animation_name` (например, `R_tg.raw.info`).

## `android_utils`

Часто используемые Android-хелперы: UI-поток, слушатели, буфер обмена, обёртки Runnable.

### `run_on_ui_thread(func, delay=0)`

Планирует Python-вызываемое на UI-потоке Android. Любая мутация UI должна происходить там: смена текста, обновление view, показ диалогов, подключение адаптеров.

```python
from android_utils import run_on_ui_thread

def update_ui_content():
    text_view = ...
    text_view.setText("Updated from Python on UI thread")

run_on_ui_thread(update_ui_content)       # как можно скорее
run_on_ui_thread(update_ui_content, 500)  # с задержкой 500 мс
```

### `R(fn)` — Java Runnable

`R(fn)` создаёт настоящий Java `Runnable` из Python-функции. Полезно для API, ожидающих `java.lang.Runnable`: `view.post(...)`, отложенные колбэки, внутренние хелперы.

```python
from android_utils import R, log

def my_task():
    log("Runnable task executed.")

runnable_instance = R(my_task)
some_view.post(runnable_instance)
```

> `run_on_ui_thread(...)` уже оборачивает ваш callable в Runnable внутри. `R(...)` нужен, когда какой-то другой API требует именно `Runnable`-объект.

### `OnClickListener(fn)`

Обёртка `android.view.View.OnClickListener`:

```python
from android_utils import OnClickListener, log
from android.view import View

def handle_button_click(view: View):
    log(f"Button {view.getId()} was clicked.")

button = ...
button.setOnClickListener(OnClickListener(handle_button_click))
```

### `OnLongClickListener(fn)`

Функция должна вернуть `bool`: `True` — перехватить событие, `False` — пустить дальше.

```python
from android_utils import OnLongClickListener, log

def handle_button_long_click(view: View):
    log(f"Button {view.getId()} was long-clicked.")
    return True

button.setOnLongClickListener(OnLongClickListener(handle_button_long_click))
```

### `copy_to_clipboard(text)`

Копирует строку в буфер обмена и показывает стандартный буллетин «скопировано»:

```python
from android_utils import copy_to_clipboard

copy_to_clipboard("https://t.me/exteraPlugins")
```

Удобно для deeplink'ов, отладочной информации, экспортируемых ID.
