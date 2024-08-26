# Шаг 4: Формы

Form classes генерируют элементы HTML для пользовательского интерфейса, а также обрабатывают и проверяют вводимые пользователем данные. Они используются в NetBox в первую очередь для создания, изменения и удаления объектов. Мы создадим класс формы для каждой из моделей нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, выполните `git checkout step03-tables`.

## Создание форм

Начнем с создания файла с именем `forms.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit forms.py
```

В верхней части файла мы импортируем класс `NetBoxModelForm` NetBox, который будет служить базовым классом для наших форм. Мы также импортируем модели нашего плагина.

```python
из netbox.forms импорт NetBoxModelForm
из .models импорт AccessList, AccessListRule
```

### AccessListForm

Создайте класс с именем `AccessListForm`, подкласс `NetBoxModelForm`. В этом классе определите подкласс `Meta`, определяющий `model` и `fields` формы. Обратите внимание, что список `fields` также включает `tags`: назначение тегов обрабатывается `NetBoxModel` автоматически, поэтому нам не нужно было добавлять его в нашу модель на втором шаге.

```python
class AccessListForm(NetBoxModelForm):

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

Этого достаточно для нашей первой модели, но мы можем внести одно изменение: вместо поля по умолчанию, которое Django сгенерирует для поля модели `comments`, мы можем использовать специально созданный класс NetBox `CommentField`. (Это обрабатывает некоторые в основном косметические детали, такие как настройка `help_text` и настройка макета поля.) Для этого просто импортируйте класс `CommentField` и переопределите поле формы:

```python
from utilities.forms.fields import CommentField
# ...
class AccessListForm(NetBoxModelForm):
    comments = CommentField()

    class Meta:
        model = AccessList
        fields = ('name', 'default_action', 'comments', 'tags')
```

### AccessListRuleForm

Мы создадим форму для `AccessListRule` по тому же шаблону.

```python
class AccessListRuleForm(NetBoxModelForm):

    class Meta:
        model = AccessListRule
        fields = (
        'access_list', 'index', 'description', 'source_prefix', 'source_ports', 'destination_prefix',
        'destination_ports', 'protocol', 'action', 'tags',
        )
```

По умолчанию Django создаст "статическое" поле внешнего ключа для связанных объектов. Оно отображается как раскрывающийся список, который предварительно заполнен _всеми_ доступными объектами. Как вы можете себе представить, в экземпляре NetBox с многими тысячами объектов это может стать довольно громоздким.

Чтобы избежать этого, NetBox предоставляет класс `DynamicModelChoiceField`. Он отображает поля внешнего ключа с помощью специального динамического виджета, поддерживаемого REST API NetBox. Это позволяет избежать накладных расходов, налагаемых статическим полем, и позволяет пользователю удобно искать нужный объект.

:green_circle: **Совет:** Класс `DynamicModelMultipleChoiceField` также доступен для полей «многие ко многим», которые поддерживают назначение нескольких объектов.

Мы будем использовать `DynamicModelChoiceField` для трех полей внешнего ключа в нашей форме: `access_list`, `source_prefix` и `destination_prefix`. Сначала мы должны импортировать класс поля, а также модели связанных объектов. `AccessList` уже импортирован, поэтому нам просто нужно импортировать `Prefix` из приложения `ipam` NetBox. Начало `forms.py` теперь должно выглядеть так:

```python
from ipam.models import Prefix
from netbox.forms import NetBoxModelForm
from utilities.forms.fields import CommentField, DynamicModelChoiceField
from .models import AccessList, AccessListRule
```

Затем мы переопределяем три соответствующих поля в классе формы, создавая экземпляр `DynamicModelChoiceField` с соответствующим значением `queryset` для каждого. (Обязательно сохраните класс `Meta`, который мы уже определили.)

```python
class AccessListRuleForm(NetBoxModelForm):
    access_list = DynamicModelChoiceField(
    queryset=AccessList.objects.all()
    )
    source_prefix = DynamicModelChoiceField(
    queryset=Prefix.objects.all()
    )
    destination_prefix = DynamicModelChoiceField(
    queryset=Prefix.objects.all()
    )
```

После того, как наши модели, таблицы и формы будут готовы, мы создадим несколько представлений, чтобы объединить все!

<div align="center">

:arrow_left: [Шаг 3: Таблицы](/tutorial/step03-tables.md) | [Шаг 5: Просмотры](/tutorial/step05-views.md) :arrow_right:

</div>