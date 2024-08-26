# Шаг 8: Наборы фильтров

Фильтры позволяют пользователям запрашивать только определенное подмножество объектов, соответствующих запросу; например, при фильтрации списка сайтов по статусу или региону. NetBox использует библиотеку [`django-filters`](https://django-filter.readthedocs.io/en/stable/) для создания и применения наборов фильтров для моделей. Мы можем создать наборы фильтров, чтобы включить эту же функциональность для нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step07-navigation`.

## Создание набора фильтров

Начнем с создания `filtersets.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit filtersets.py
```

В верхней части этого файла мы импортируем класс `NetBoxModelFilterSet` NetBox, который будет служить базовым классом для нашего набора фильтров, а также для нашей модели `AccessListRule`. (Для краткости мы создадим набор фильтров только для одной модели, но должно быть ясно, как повторить этот подход и для модели `AccessList`.)

```python
from netbox.filtersets import NetBoxModelFilterSet
from .models import AccessListRule
```

Далее создайте класс с именем `AccessListRuleFilterSet`, подклассифицирующий `NetBoxModelFilterSet`. В этом классе создайте дочерний класс `Meta` и определите атрибуты `model` и `fields` набора фильтров. (Вы можете заметить, что это выглядит знакомо; это очень похоже на процесс создания формы модели.) Параметр `fields` должен содержать список всех полей модели, по которым мы можем захотеть выполнить фильтрацию.

```python
class AccessListRuleFilterSet(NetBoxModelFilterSet):

    class Meta:
        model = AccessListRule
        fields = ('id', 'access_list', 'index', 'protocol', 'action')
```

`NetBoxModelFilterSet` обрабатывает некоторые важные для нас функции, включая поддержку фильтрации по значениям пользовательских полей и тегам. Он также создает универсальный фильтр `q`, который вызывает метод `search()`. (По умолчанию он ничего не делает.) Мы можем переопределить этот метод, чтобы определить нашу универсальную логику поиска. Давайте добавим метод `search` после дочернего класса `Meta`, чтобы переопределить поведение по умолчанию.

```python
def search(self, queryset, name, value):
    return queryset.filter(description__icontains=value)
```

Это вернет все правила, описание которых содержит запрашиваемую строку. Конечно, вы можете расширить это, чтобы соответствовать и другим полям, но для наших целей этого должно быть достаточно.

## Создание формы фильтра

Набор фильтров обрабатывает «закулисный» процесс фильтрации запросов, но нам также нужно создать класс формы для отображения полей фильтра в пользовательском интерфейсе. Мы добавим это в `forms.py`. Сначала импортируйте модуль `forms` Django (который предоставит необходимые нам классы полей) и добавьте `NetBoxModelFilterSetForm` к существующему оператору импорта для `netbox.forms`:

```python
from django import forms
# ...
from netbox.forms import NetBoxModelForm, NetBoxModelFilterSetForm
```

Затем создайте класс формы с именем `AccessListRuleFilterForm`, подкласс `NetBoxModelFilterSetForm` и объявите атрибут с именем `model`, ссылающийся на `AccessListRule` (который уже был импортирован для одной из существующих форм).

```python
class AccessListRuleFilterForm(NetBoxModelFilterSetForm):
    model = AccessListRule
```

:blue_square: **Примечание:** Обратите внимание, что атрибут `model` объявлен непосредственно под классом: нам не нужен дочерний класс `Meta`.

Далее нам нужно определить поле формы для каждого фильтра, который мы хотим отобразить в пользовательском интерфейсе. Начнем с фильтра `access_list`: он ссылается на связанный объект, поэтому мы захотим использовать `ModelMultipleChoiceField` (чтобы пользователи могли фильтровать по нескольким объектам). Добавьте поле формы с тем же именем, что и у его фильтра-одноранговца, указав набор запросов, который будет использоваться при извлечении связанных объектов.


```python
    access_list = forms.ModelMultipleChoiceField(
        queryset=AccessList.objects.all(),
        required=False
    )
```

Обратите внимание, что мы также установили `required=False`: это должно быть применимо ко всем полям в форме фильтра, поскольку фильтры никогда не являются обязательными.

:blue_square: **Примечание:** Мы используем класс `ModelMultipleChoiceField` Django для этого поля вместо `DynamicModelChoiceField` NetBox, поскольку последний требует функциональной конечной точки REST API для модели. После того, как мы реализуем REST API на девятом шаге, вы можете вернуться к этой форме и изменить `access_list` на `DynamicModelChoiceField`.

Далее мы добавим поле для фильтра `position`: Это целочисленное поле, поэтому `IntegerField` должно работать хорошо:

```python
    index = forms.IntegerField(
        required=False
    )
```

Наконец, мы добавим поля для фильтров на основе выбора `protocol` и `action`. `MultipleChoiceField` следует использовать, чтобы пользователи могли выбрать один или несколько вариантов. Мы должны передать набор допустимых вариантов при объявлении этих полей, поэтому сначала расширьте соответствующий оператор импорта в верхней части `forms.py`:

```python
from .models import AccessList, AccessListRule, ActionChoices, ProtocolChoices
```

Затем добавьте поля формы в `AccessListRuleFilterForm`:

```python
    protocol = forms.MultipleChoiceField(
        choices=ProtocolChoices,
        required=False
    )
    action = forms.MultipleChoiceField(
        choices=ActionChoices,
        required=False
    )
```

## Обновите представление

Последний шаг перед тем, как мы сможем использовать наш новый набор фильтров и форму, — включить их в представлении списка модели. Откройте `views.py` и расширьте последний оператор импорта, включив модуль `filtersets`:

```python
from . import filtersets, forms, models, tables
```

Затем добавьте атрибуты `filterset` и `filterset_form` в `AccessListRuleListView`:

```python
class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable
    filterset = filtersets.AccessListRuleFilterSet
    filterset_form = forms.AccessListRuleFilterForm
```

Убедившись, что сервер разработки перезапущен, перейдите к представлению списка правил в браузере. Теперь вы должны увидеть вкладку «Фильтры» рядом с вкладкой «Результаты». Под ней мы найдем четыре поля, которые мы создали в `AccessListRuleFilterForm`, а также встроенное поле «поиска».

![Форма фильтра правил списка доступа](/images/step08-filter-form.png)

Если вы еще этого не сделали, создайте еще несколько списков доступа и правил и поэкспериментируйте с фильтрами. Подумайте, как можно фильтровать по дополнительным полям или добавить более сложную логику в набор фильтров.

:green_circle: **Совет:** Вы можете заметить, что мы не добавили поле формы для фильтра `id` модели: это потому, что оно вряд ли будет полезно для человека, использующего пользовательский интерфейс. Однако мы все равно хотим поддерживать фильтрацию объектов по их первичным ключам, потому что это _очень_ полезно для потребителей REST API NetBox, о чем мы поговорим далее.

<div align="center">

:arrow_left: [Шаг 7: Навигация](/tutorial/step07-navigation.md) | [Шаг 9: REST API](/tutorial/step09-rest-api.md) :arrow_right:

</div>