# Шаг 7: Навигация

До сих пор мы вручную вводили URL-адреса для доступа к представлениям нашего плагина. Очевидно, что этого будет недостаточно для обычного использования, поэтому давайте посмотрим, как добавить несколько ссылок в меню навигации NetBox.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step06-templates`.

## Добавление элементов меню навигации

Начнем с создания `navigation.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit navigation.py
```

Нам нужно будет импортировать класс `PluginMenuItem`, предоставляемый NetBox, чтобы добавить новые элементы меню; сделайте это в верхней части файла.

```python
from extras.plugins import PluginMenuItem
```

Далее мы создадим кортеж с именем `menu_items`. Он будет содержать наши настроенные экземпляры `PluginMenuItem`.

```python
menu_items = ()
```

Давайте добавим ссылку на представление списка для каждой из наших моделей. Это делается путем создания экземпляра `PluginMenuItem` с (минимум) двумя аргументами:

* `link` - Имя URL-пути, на который мы ссылаемся
* `link_text` - Текст ссылки

Создайте два экземпляра `PluginMenuItem` в `menu_items`:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists'
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules'
    ),
)
```

После перезагрузки страницы вы должны увидеть новый раздел "Плагины" в конце меню навигации, а под ним раздел под названием «Списки доступа NetBox» с нашими двумя ссылками. Переход по любой из этих ссылок выделит соответствующий пункт меню.

:blue_square: **Примечание:** Если пункты меню не отображаются, попробуйте перезапустить сервер разработки (`manage.py runserver`).

![Пункты меню навигации](/images/step07-menu-items1.png)

Это гораздо удобнее!

### Добавление кнопок меню

Пока мы этим занимаемся, мы можем добавить прямые ссылки на представления «добавить» для списков доступа и правил в качестве кнопок. Нам нужно будет импортировать два дополнительных класса в верхней части `navigation.py`: `PluginMenuButton` и `ButtonColorChoices`.

```python
from extras.plugins import PluginMenuButton, PluginMenuItem
from utilities.choices import ButtonColorChoices
```

`PluginMenuButton` используется аналогично `PluginMenuItem`: создайте его с необходимыми ключевыми аргументами, чтобы сделать кнопку меню. Эти аргументы:

* `link` - Имя URL-пути, на который ссылается кнопка
* `title` - Текст, отображаемый при наведении курсора на кнопку
* `icon_class` - Имя(а) класса CSS, указывающие значок для отображения
* `color` - Цвет кнопки (выборы предоставляются `ButtonColorChoices`)

Создайте эти экземпляры в `navigation.py` _выше_ `menu_items`. Поскольку каждый элемент меню ожидает получить итерируемый набор экземпляров кнопок, мы создадим каждый из них внутри списка.

```python
accesslist_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslist_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]

accesslistrule_buttons = [
    PluginMenuButton(
        link='plugins:netbox_access_lists:accesslistrule_add',
        title='Add',
        icon_class='mdi mdi-plus-thick',
        color=ButtonColorChoices.GREEN
    )
]
```

Затем кнопки можно передать в пункты меню с помощью ключевого аргумента `buttons`:

```python
menu_items = (
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslist_list',
        link_text='Access Lists',
        buttons=accesslist_buttons
    ),
    PluginMenuItem(
        link='plugins:netbox_access_lists:accesslistrule_list',
        link_text='Access List Rules',
        buttons=accesslistrule_buttons
    ),
)
```

Теперь мы должны увидеть кнопки «добавить» рядом со ссылками нашего меню.

![Пункты меню навигации с кнопками](/images/step07-menu-items2.png)

<div align="center">

:arrow_left: [Шаг 6: Шаблоны](/tutorial/step06-templates.md) | [Шаг 8: Наборы фильтров](/tutorial/step08-filter-sets.md) :arrow_right:

</div>