# Шаг 3: Таблицы

Вы, вероятно, знакомы со списками объектов в NetBox. Так мы отображаем все экземпляры определенного типа объекта, например, сайтов или устройств, в пользовательском интерфейсе. Эти списки генерируются классами таблиц, определенными для каждой модели, с использованием библиотеки [django-tables2](https://django-tables2.readthedocs.io/).

Хотя было бы возможно генерировать необработанный HTML для элементов `<table>` непосредственно в шаблоне, это было бы громоздко и сложно в обслуживании. Кроме того, эти динамические классы таблиц предоставляют удобные функции, такие как сортировка и разбиение на страницы.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step02-models`.

## Создание таблиц

Мы создадим две таблицы, по одной для каждой из наших моделей. Начните с создания `tables.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit tables.py
```

В верхней части этого файла импортируйте библиотеку `django-tables2`. Она предоставит классы столбцов для полей, которые мы хотим настроить. Мы также импортируем класс `NetBoxTable` NetBox, который будет служить базовым классом для наших таблиц, и `ChoiceFieldColumn`. Наконец, мы импортируем модели нашего плагина из `models.py`.

```python
import django_tables2 as tables

from netbox.tables import NetBoxTable, ChoiceFieldColumn
from .models import AccessList, AccessListRule
```

### AccessListTable

Создайте класс с именем `AccessListTable` как подкласс `NetBoxTable`. В этом классе создайте дочерний класс `Meta`, унаследованный от `NetBoxTable.Meta`; он определит модель, поля и столбцы по умолчанию таблицы.

```python
class AccessListTable(NetBoxTable):

    class Meta(NetBoxTable.Meta):
        model = AccessList
        fields = ('pk', 'id', 'name', 'default_action', 'comments', 'actions')
        default_columns = ('name', 'default_action')
```

Атрибут `model` сообщает `django-tables2`, какую модель использовать при построении таблицы, а атрибут `fields` определяет, какие поля модели будут добавлены в таблицу. `default_columns` управляет тем, какие из доступных столбцов отображаются по умолчанию.

Столбцы `pk` и `actions` отображают селекторы флажков и раскрывающиеся меню соответственно для каждой строки таблицы; они предоставляются классом NetBoxTable. Столбец `id` будет отображать числовой первичный ключ объекта, который включен почти в каждую таблицу в NetBox, но обычно отключен по умолчанию. Остальные три столбца выводятся из полей, которые мы определили в модели `AccessList`.

То, что у нас есть на данный момент, достаточно для отображения таблицы, но мы можем внести некоторые небольшие улучшения. Сначала сделаем столбец `name` ссылкой на каждый объект. Для этого мы переопределим столбец по умолчанию, определив `name` в классе и передав `linkify=True`.

```python
class AccessListTable(NetBoxTable):
    name = tables.Column(
        linkify=True
)
```

Кроме того, напомним, что поле `default_action` в модели `AccessList` является полем выбора, с цветом, назначенным для каждого выбора. Для отображения этих значений мы будем использовать класс `ChoiceFieldColumn` NetBox.

```python
    default_action = ChoiceFieldColumn()
```

Также было бы неплохо включить счетчик, показывающий количество правил, назначенных каждому списку доступа. Мы можем добавить пользовательский столбец с именем `rule_count`, чтобы показать это. (Данные для этого столбца будут аннотированы представлением; подробнее об этом в шаге пять.) Нам также нужно будет добавить этот столбец в наши `fields` и (необязательно) `default_columns` в подклассе `Meta`. Наша готовая таблица должна выглядеть так:

```python
class AccessListTable(NetBoxTable):
    name = tables.Column(
        linkify=True
    )
    default_action = ChoiceFieldColumn()
    rule_count = tables.Column()

    class Meta(NetBoxTable.Meta):
        model = AccessList
        fields = ('pk', 'id', 'name', 'rule_count', 'default_action', 'comments', 'actions')
        default_columns = ('name', 'rule_count', 'default_action')
```

### AccessListRuleTable

Мы также создадим таблицу для нашей модели `AccessListRule`, используя тот же подход, что и выше. Начните с связывания столбцов `access_list` и `index`. Первый будет ссылаться на родительский список доступа, а второй — на индивидуальное правило. Мы также хотим объявить `protocol` и `action` как экземпляры `ChoiceFieldColumn`.

```python
class AccessListRuleTable(NetBoxTable):
    access_list = tables.Column(
        linkify=True
    )
    index = tables.Column(
        linkify=True
    )
    protocol = ChoiceFieldColumn()
    action = ChoiceFieldColumn()

    class Meta(NetBoxTable.Meta):
        model = AccessListRule
        fields = (
            'pk', 'id', 'access_list', 'index', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'description', 'actions',
        )
        default_columns = (
            'access_list', 'index', 'source_prefix', 'source_ports', 'destination_prefix',
            'destination_ports', 'protocol', 'action', 'actions',
        )
```

Это все, что нам нужно для перечисления этих объектов в пользовательском интерфейсе. Далее мы определим некоторые формы, чтобы разрешить создание и изменение объектов.

<div align="center">

:arrow_left: [Шаг 2: Модели](/tutorial/step02-models.md) | [Шаг 4: Формы](/tutorial/step04-forms.md) :arrow_right:

</div>
