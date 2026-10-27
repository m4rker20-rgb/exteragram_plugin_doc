# 03. Настройки плагина

Экран настроек, подстраницы и кастомные элементы UI.

Экран настроек строится методом `create_settings(self) -> List[Any]`, который возвращает список дата-классов из `ui.settings`.

## Общий пример

Пример почти со всеми встроенными типами элементов:

```python
from typing import Any, List

from android.view import Gravity, View
from android.widget import LinearLayout

from org.telegram.messenger import AndroidUtilities, MediaDataController, UserConfig
from org.telegram.ui.Components import BackupImageView, LayoutHelper, UItem

from base_plugin import BasePlugin
from ui.settings import (
    Custom, Divider, EditText, Header, Input, Selector, SimpleSettingFactory, Switch, Text,
)


def create_sticker_view(context, list_view, current_account: int, class_guid: int, resources_provider):
    main_layout = LinearLayout(context)
    main_layout.setOrientation(LinearLayout.VERTICAL)
    main_layout.setGravity(Gravity.CENTER)
    main_layout.setPadding(AndroidUtilities.dp(20), AndroidUtilities.dp(20), AndroidUtilities.dp(20), AndroidUtilities.dp(20))

    image_view = BackupImageView(context)
    image_view.setRoundRadius(AndroidUtilities.dp(45))

    # View создаётся один раз. Данные не привязываем здесь.
    main_layout.addView(image_view, LayoutHelper.createLinear(130, 130, Gravity.CENTER, 0, 0, 0, 16))
    return main_layout


def bind_sticker_view(view, item, divider: bool, adapter, list_view):
    # Вызывается, когда view должен показать реальные данные.
    MediaDataController.getInstance(UserConfig.selectedAccount).setPlaceholderImageByIndex(
        view.getChildAt(0), "CactusPlugins", 0, "130_130",
    )


StickerSettingFactory = SimpleSettingFactory(
    create_sticker_view, bind_sticker_view, is_clickable=False, is_shadow=False,
)


class MyPlugin(BasePlugin):
    def _on_test_switch_change(self, new_value: bool):
        self.logger.info(f"Switch changed to: {new_value}")

    def _on_test_input_change(self, new_value: str):
        self.logger.info(f"Input changed to: {new_value}")

    def _on_test_selector_change(self, new_index: int):
        self.logger.info(f"Selector changed to index: {new_index}")

    def _create_sub_page(self) -> List[Any]:
        return [
            Header(text="This is a Sub-Page"),
            Text(text="You can nest settings pages."),
        ]

    def create_settings(self) -> List[Any]:
        return [
            Header(text="General Settings"),
            Switch(
                key="test_switch_key", text="Test Switch", default=True,
                subtext="Sample switch control.", icon="msg_settings",
                on_change=self._on_test_switch_change, link_alias="test_switch",
            ),
            Selector(
                key="test_selector_key", text="Test Selector", default=1,
                items=["Option A", "Option B", "Option C"], icon="msg_list",
                on_change=self._on_test_selector_change,
            ),
            Input(
                key="test_input_key", text="Test Input", default="Hello, World!",
                subtext="Single-line text input.", icon="msg_text",
                on_change=self._on_test_input_change,
            ),
            EditText(key="multiline_key", hint="Enter multiple lines...", default="", multiline=True, max_length=1000),
            Divider(text="This is a divider with text."),
            Text(
                text="Click for Sub-Page", icon="msg_arrow_forward",
                on_click=lambda view: None,
                create_sub_fragment=self._create_sub_page, link_alias="sub_page_link",
            ),
            Text(text="This is red text", icon="msg_error", red=True),
            Divider(),
            Header(text="Custom Controls"),
            Custom(factory=StickerSettingFactory.instance.java),
            Custom(item=UItem.asShadow("This is a custom UItem shadow.")),
        ]
```

## Чтение и запись значений

```python
# Прочитать значение с fallback:
is_enabled = self.get_setting("test_switch_key", False)

# Записать значение:
current_value = self.get_setting("test_switch_key", False)
self.set_setting("test_switch_key", not current_value)

# Если изменение должно перестроить весь открытый экран настроек:
self.set_setting("main_option", "A", reload_settings=True)
```

Экспорт/импорт всех настроек плагина:

```python
all_my_settings = self.export_settings()
self.logger.info(f"My settings: {all_my_settings}")

new_settings = {"test_switch_key": False, "test_input_key": "New Value"}
self.import_settings(new_settings)                    # UI перезагрузится
self.import_settings(new_settings, reload_settings=False)  # без перезагрузки UI
```

## Встроенные типы элементов

Публичные дата-классы из `ui.settings`: `Header`, `Divider`, `Switch`, `Selector`, `Input`, `Text`, `EditText`, `Custom`.

| Контрол | `key` | `text` | `default` | Прочие важные параметры |
|---|---|---|---|---|
| `Header` | — | обязательнo | — | `text`: заголовок секции |
| `Divider` | — | — | — | `text`: необязательная подпись |
| `Switch` | ✅ | ✅ | ✅ (`bool`) | `subtext`, `icon`, `on_change`, `on_long_click`, `link_alias` |
| `Selector` | ✅ | ✅ | ✅ (`int`) | `items`, `icon`, `on_change`, `on_long_click`, `link_alias` |
| `Input` | ✅ | ✅ | опц. `str` | `subtext`, `icon`, `on_change`, `on_long_click`, `link_alias` |
| `Text` | — | ✅ | — | `subtext`, `icon`, `accent`, `red`, `on_click`, `on_long_click`, `create_sub_fragment`, `link_alias` |
| `EditText` | ✅ | — | опц. `str` | `hint` (обязателен), `multiline`, `max_length`, `mask`, `on_change` |
| `Custom` | — | — | — | `item`, `view`, `factory`, `factory_args`, `on_click`, `on_long_click`, `create_sub_fragment`, `link_alias` |

### Детали параметров

| Параметр | Тип | Описание |
|---|---|---|
| `key` | `str` | Уникальный ключ для `get_setting()` / `set_setting()` |
| `text` | `str` | Основной текст строки |
| `default` | `Any` | Значение по умолчанию, если ничего не сохранено |
| `subtext` | `str` | Вторая строка |
| `icon` | `str` | Имя drawable, например `"msg_settings"` |
| `on_change` | `Callable` | Вызывается при изменении значения; сигнатура зависит от типа |
| `on_click` | `Callable` | Вызывается при нажатии на строку (обычно получает `View`) |
| `on_long_click` | `Callable` | Долгое нажатие |
| `link_alias` | `str` | Включает копируемый deeplink-алиас для строки |
| `items` | `List[str]` | Подписи вариантов `Selector` |
| `create_sub_fragment` | `Callable[[], List[Any]]` | Строит вложенную страницу настроек |
| `accent` / `red` | `bool` | Цветовая раскраска `Text` |
| `hint` | `str` | Плейсхолдер `EditText` |
| `multiline` | `bool` | Многострочность `EditText` |
| `max_length` | `int` | Максимальная длина ввода |
| `mask` | `str` | Regex-подобное ограничение ввода |
| `factory` | `CustomSetting.Factory` | Java-фабрика для сложных кастомных элементов |
| `item` | `UItem` | Готовый Telegram `UItem` |
| `view` | `View` | Обычный Android `View` |

## Кастомные элементы (`Custom`)

`Custom` — точка входа для строк, не покрывающихся встроенными контролами. Три способа:

### 1. Готовый `UItem`

Самый лёгкий вариант, когда в Telegram уже есть подходящий тип строки:

```python
from org.telegram.ui.Components import UItem
from ui.settings import Custom

setting = Custom(item=UItem.asShadow("This is a custom UItem shadow."))
```

### 2. Готовый Android `View`

Для статичных или уже созданных view:

```python
from android.widget import TextView
from ui.settings import Custom

label = TextView(context)
label.setText("Hello from a custom View")

setting = Custom(view=label)
```

Подходит для простых одноразовых view. Если строке нужны рециклинг, привязка данных или клики — используйте фабрику.

### 3. `SimpleSettingFactory` (рекомендуется)

Высокоуровневый API: внутри создаёт Java `CustomSetting.Factory`, обрабатывает биндинг и даёт Python-колбэки.

```python
from android.view import Gravity
from android.widget import LinearLayout, TextView

from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.Components import LayoutHelper

from ui.bulletin import BulletinHelper
from ui.settings import Custom, SimpleSettingFactory


def create_view(context, list_view, current_account, class_guid, resources_provider):
    main_layout = LinearLayout(context)
    main_layout.setOrientation(LinearLayout.VERTICAL)
    main_layout.setGravity(Gravity.CENTER)
    main_layout.setPadding(AndroidUtilities.dp(20), AndroidUtilities.dp(20), AndroidUtilities.dp(20), AndroidUtilities.dp(20))

    title = TextView(context)
    title.setText("Tap me")
    main_layout.addView(title, LayoutHelper.createLinear(LayoutHelper.WRAP_CONTENT, LayoutHelper.WRAP_CONTENT))
    return main_layout


def bind_view(view, item, divider: bool, adapter, list_view):
    view.getChildAt(0).setText("Tap me")


def on_click(plugin, item, view):
    BulletinHelper.show_info(f"Hello from {plugin.getName()}!")


InteractiveRowFactory = SimpleSettingFactory(create_view, bind_view, is_clickable=True, on_click=on_click)

setting = Custom(factory=InteractiveRowFactory.instance.java)
```

#### Колбэки фабрики

- `create_view(context, list_view, current_account, class_guid, resources_provider) -> View` — строит view строки;
- `bind_view(view, item, divider, adapter, list_view)` — заполняет текущими данными;
- `create_item(plugin, setting, args) -> UItem`;
- `on_click(plugin, item, view)`;
- `on_long_click(plugin, item, view) -> bool`;
- `attached_view(list_view, view, item)`;
- `equals(a, b) -> bool`, `content_equals(a, b) -> bool`.

#### Конструктор

```python
SimpleSettingFactory(
    create_view,
    bind_view,
    is_clickable=False,
    is_shadow=False,
    create_item=None,
    on_click=None,
    on_long_click=None,
    attached_view=None,
    equals=None,
    content_equals=None,
)
```

- `is_clickable` и `is_shadow` — Java-флаги, оптимизированные в горячих путях;
- если `create_item` не задан, фабрика создаёт `UItem` по умолчанию;
- если `equals`/`content_equals` не заданы — стандартная логика сравнения `UItem`.

#### Фабрика как вызываемый объект

`SimpleSettingFactory` реализует `__call__`, поэтому работает как строитель для `Custom(...)`:

```python
StickerFactory = SimpleSettingFactory(create_sticker_view, bind_sticker_view)

setting = StickerFactory(link_alias="sticker_preview")
setting = StickerFactory("CactusPlugins", 0)  # → factory_args
```

#### Жизненный цикл фабрики

Создавайте фабрику **один раз** (например, на уровне модуля) и переиспользуйте при каждом вызове `create_settings()`. Фабрика остаётся зарегистрированной вместе с переданными функциями до выгрузки плагина — создание новых экземпляров при каждом построении настроек приводит к росту памяти.

Временную фабрику можно освободить явно:

```python
factory = SimpleSettingFactory(create_view, bind_view)
...
factory.release()
```

## Продвинутые фабрики

Если `SimpleSettingFactory` мало — сгенерируйте собственный подкласс `CustomSetting.Factory` через **class proxy** (реальный Java-подкласс, свои поля, `super()`, MVEL-методы). См. [10. Class Proxy](10-class-proxy.md) — там есть готовый пример фабрики с ячейками списка плагинов.

## Связанные темы

- [11. Elyx](11-elyx-structured-plugins.md) — в Elyx-плагинах значения читаются через `from elyx import settings`
- [07. Android-утилиты](07-android-utils-and-logging.md) — `run_on_ui_thread` для потокобезопасных изменений UI
