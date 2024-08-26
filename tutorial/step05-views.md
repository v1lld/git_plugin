# Шаг 5: Представления

Представления отвечают за бизнес-логику вашего приложения. Обычно это означает обработку входящих запросов, выполнение некоторых действий и возврат ответа клиенту. Каждое представление обычно имеет связанный с ним URL и может обрабатывать один или несколько типов HTTP-запросов (например, запросы `GET` и/или `POST`).

Django предоставляет набор [универсальных классов представлений](https://docs.djangoproject.com/en/4.0/topics/class-based-views/generic-display/), которые обрабатывают большую часть шаблонного кода, необходимого для обработки запросов. NetBox также предоставляет набор классов представлений для упрощения создания представлений для создания, редактирования, удаления и просмотра объектов. Они также вводят поддержку специфичных для NetBox функций, таких как настраиваемые поля и ведение журнала изменений.

На этом этапе мы создадим набор представлений для каждой из моделей нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step04-forms`.

## Создание представлений

Начнем с создания `views.py` в каталоге `netbox_access_lists/`.

```bash
$ cd netbox_access_lists/
$ edit views.py
```

Нам нужно будет импортировать модули `models`, `tables` и `forms` нашего плагина: вот где все, что мы создали до сих пор, действительно объединяется! Нам также нужно импортировать модуль generic views NetBox, так как он предоставляет базовые классы для наших представлений.

```python
from netbox.views import generic
from . import forms, models, tables
```

:green_circle: **Совет:** Вы заметите, что мы импортируем здесь все модули model, form и tables. Если вы предпочитаете импортировать каждый из соответствующих классов напрямую, вы, конечно, можете это сделать; просто не забудьте изменить определения классов ниже соответствующим образом.

Для каждой модели нам нужно создать четыре представления:

* **Подробное представление** — Отображение одного объекта
* **Представление списка** — Отображение таблицы всех существующих экземпляров определенной модели
* **Представление редактирования** — Обработка добавления и изменения объектов
* **Представление удаления** — Обработка удаления объекта

### Представления списка доступа

Общий шаблон, которому мы будем следовать здесь, заключается в подклассификации универсального класса представления, предоставляемого NetBox, и определении необходимых атрибутов. Нам не нужно будет писать какой-либо существенный код, поскольку представления, предоставляемые NetBox, заботятся о логике запроса за нас.

Давайте начнем с подробного представления. Мы подклассифицируем `generic.ObjectView` и определяем набор запросов объектов, которые мы хотим отобразить.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()
```

:green_circle: **Совет:** Для представлений требуется определить набор запросов, а не просто модель, поскольку иногда необходимо изменить набор запросов, например, для предварительной выборки связанных объектов или ограничения по определенному атрибуту.

Далее мы добавим представление списка. Для этого представления нам нужно определить как `queryset`, так и `table`.

```python
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.all()
    table = tables.AccessListTable
```

:green_circle: **Совет:** Автору приходит в голову, что выбор имени модели, заканчивающегося на "List", может немного сбить с толку. Просто помните, что `AccessListView` — это _детальное_ (отдельный объект) представление, а `AccessListListView` — это _списочное_ (несколько объектов) представление.

Прежде чем перейти к следующему представлению, помните ли вы дополнительный столбец, который мы добавили в `AccessListTable` на третьем шаге? Этот столбец ожидает найти количество правил, назначенных для каждого списка доступа в наборе запросов, с именем `rule_count`. Давайте добавим это в наш набор запросов сейчас. Мы можем использовать функцию `Count()` Django, чтобы расширить SQL-запрос и аннотировать количество связанных правил. (Не забудьте добавить оператор импорта вверху.)

```python
from django.db.models import Count
# ...
class AccessListListView(generic.ObjectListView):
    queryset = models.AccessList.objects.annotate(
    rule_count=Count('rules')
    )
    table = tables.AccessListTable
```

Мы закончим с представлениями редактирования и удаления для `AccessList`. Обратите внимание, что для представления редактирования нам также нужно определить `form` как класс формы, который мы создали на четвертом шаге.

```python
class AccessListEditView(generic.ObjectEditView):
    queryset = models.AccessList.objects.all()
    form = forms.AccessListForm

class AccessListDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessList.objects.all()
```

Вот и все для первой модели! Мы также создадим еще четыре представления для `AccessListRule`.

### Представления AccessListRule

Остальные наши представления следуют тому же шаблону, что и первые четыре.

```python
class AccessListRuleView(generic.ObjectView):
    queryset = models.AccessListRule.objects.all()


class AccessListRuleListView(generic.ObjectListView):
    queryset = models.AccessListRule.objects.all()
    table = tables.AccessListRuleTable


class AccessListRuleEditView(generic.ObjectEditView):
    queryset = models.AccessListRule.objects.all()
    form = forms.AccessListRuleForm


class AccessListRuleDeleteView(generic.ObjectDeleteView):
    queryset = models.AccessListRule.objects.all()
```

Теперь, когда наши представления готовы, нам нужно сделать их доступными, связав каждое с URL-адресом.

## Сопоставьте представления с URL-адресами

В каталоге `netbox_access_lists/` создайте `urls.py`. Это определит наши URL-адреса представлений.

```bash
$ edit urls.py
```

Сопоставление URL-адресов для плагинов NetBox практически идентично обычным приложениям Django: мы определим `urlpatterns` как итерируемый набор вызовов `path()`, сопоставляя фрагменты URL-адресов с классами представлений.

Сначала нам нужно импортировать функцию `path` Django из его модуля `urls`, а также модули `models` и `views` нашего плагина.

```python
from django.urls import path
from . import models, views
```

У нас есть четыре представления на модель, но на самом деле нам нужно определить пять путей для каждого. Это связано с тем, что операции добавления и редактирования обрабатываются одним и тем же представлением, но доступ к ним осуществляется через разные URL-адреса. Вместе с URL-адресом и представлением для каждого пути мы также укажем `name`; это позволяет нам легко ссылаться на URL-адрес в коде.

```python
urlpatterns = (
path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),
)
```

Мы выбрали `access-lists` в качестве базового URL для нашей модели `AccessList`, но вы можете выбрать что-то другое. Однако рекомендуется сохранить показанную схему именования, так как на нее опираются несколько функций NetBox. Также обратите внимание, что каждое из представлений должно вызываться его методом `as_view()` при передаче в `path()`.

:green_circle: **Совет:** Строка `<int:pk>`, которую вы видите в некоторых URL, является [преобразователем пути](https://docs.djangoproject.com/en/stable/topics/http/urls/#path-converters). В частности, это целочисленная (`int`) переменная с именем `pk`. Это значение извлекается из URL запроса и передается в представление при обработке запроса, чтобы указанный объект можно было найти в базе данных.

Давайте добавим остальные пути. Возможно, вам будет полезно разделить пути по моделям, чтобы сделать файл более читабельным.

```python
urlpatterns = (

# Списки доступа
path('access-lists/', views.AccessListListView.as_view(), name='accesslist_list'),
path('access-lists/add/', views.AccessListEditView.as_view(), name='accesslist_add'),
path('access-lists/<int:pk>/', views.AccessListView.as_view(), name='accesslist'),
path('access-lists/<int:pk>/edit/', views.AccessListEditView.as_view(), name='accesslist_edit'),
path('access-lists/<int:pk>/delete/', views.AccessListDeleteView.as_view(), name='accesslist_delete'),

# Правила списка доступа
path('rules/', views.AccessListRuleListView.as_view(), name='accesslistrule_list'),
path('rules/add/', views.AccessListRuleEditView.as_view(), name='accesslistrule_add'),
path('rules/<int:pk>/', views.AccessListRuleView.as_view(), name='accesslistrule'),
path('rules/<int:pk>/edit/', views.AccessListRuleEditView.as_view(), name='accesslistrule_edit'),
path('rules/<int:pk>/delete/', views.AccessListRuleDeleteView.as_view(), name='accesslistrule_delete'),

)
```

### Добавление представлений журнала изменений

Вы можете вспомнить, что одной из функций, предоставляемых NetBox, является автоматическое [ведение журнала изменений](https://netbox.readthedocs.io/en/stable/additional-features/change-logging/). Вы можете увидеть это в действии, просматривая объект NetBox и выбирая его вкладку «Журнал изменений». Поскольку наши модели наследуются от `NetBoxModel`, они также могут использовать эту функцию.

Мы добавим выделенный URL-адрес журнала изменений для каждой из наших моделей. Сначала, вернувшись к началу `urls.py`, нам нужно импортировать `ObjectChangeLogView` NetBox:

```python
from netbox.views.generic import ObjectChangeLogView
```

Затем мы добавим дополнительный путь для каждой модели внутри `urlpatterns`:

```python
urlpatterns = (

# Списки доступа
# ...
path('access-lists/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslist_changelog', kwargs={
'model': models.AccessList
}),

# Правила списков доступа
# ...
path('rules/<int:pk>/changelog/', ObjectChangeLogView.as_view(), name='accesslistrule_changelog', kwargs={
'model': models.AccessListRule
}),

)
```

Обратите внимание, что мы используем `ObjectChangeLogView` напрямую; нам не нужно было создавать для него подклассы, специфичные для модели. Кроме того, мы передаем ключевой аргумент `model` в представление: это указывает модель, которая будет использоваться (именно поэтому нам не нужно было создавать подкласс представления).

## Добавить методы URL модели

Теперь, когда у нас есть пути URL, мы можем добавить метод `get_absolute_url()` к каждой из наших моделей. Этот метод является [соглашением Django](https://docs.djangoproject.com/en/stable/ref/models/instances/#get-absolute-url); хотя это и не является строго обязательным, он удобно возвращает абсолютный URL для любого конкретного объекта. Например, вызов `accesslist.get_absolute_url()` вернет `/plugins/access-lists/access-lists/123/` (где 123 — первичный ключ объекта).

Вернитесь в `models.py`, импортируйте функцию `reverse` Django из ее модуля `urls` в верхней части файла:

```python
from django.urls import reverse
```

Затем добавьте метод `get_absolute_url()` в класс `AccessList` после его метода `__str__()`:

```python
class AccessList(NetBoxModel):
# ...
def get_absolute_url(self):
    return reverse('plugins:netbox_access_lists:accesslist', args=[self.pk])
```

`reverse()` принимает здесь два аргумента: имя представления и список позиционных аргументов. Имя представления формируется путем объединения трех компонентов:

* Строка `'plugins'`
* Имя нашего плагина
* Имя желаемого пути URL (определенного как `name='accesslist'` в `urls.py`)

Также передается атрибут `pk` объекта, который заменяет преобразователь пути `<int:pk>` в URL.

Мы также добавим метод `get_absolute_url()` для `AccessListRule`, соответствующим образом изменив имя представления.

```python
class AccessListRule(NetBoxModel):
# ...
def get_absolute_url(self):
    return reverse('plugins:netbox_access_lists:accesslistrule', args=[self.pk])
```

## Тестирование представлений

Теперь момент истины: вся ли наша работа до сих пор привела к функциональным представлениям пользовательского интерфейса? Проверьте, что сервер разработки запущен, затем откройте браузер и перейдите по адресу <http://localhost:8000/plugins/access-lists/access-lists/>. Вы должны увидеть представление списка доступа и (если вы следовали шагу два) один список доступа с именем MyACL1.

:blue_square: **Примечание:** В этом руководстве предполагается, что вы запускаете сервер разработки Django локально на порту 8000. Если ваши настройки отличаются, вам нужно будет соответствующим образом настроить ссылку выше.

![Представление списка списков доступа](/images/step05-accesslist-list.png)

Мы видим, что наша таблица успешно отобразила столбцы `name`, `rule_count` и `default_action`, которые мы определили на шаге три, а столбец `rule_count` показывает два правила, назначенных, как и ожидалось.

Если мы нажмем кнопку «Добавить» вверху справа, мы перейдем к форме создания списка доступа. (Создание нового списка доступа пока не работает, но форма должна отображаться так, как показано ниже.)

![Форма создания списка доступа](/images/step05-accesslist-form.png)

Однако, если вы нажмете на ссылку на список доступа в таблице, вы увидите исключение `TemplateDoesNotExist`. Это означает именно то, что оно говорит: мы еще не определили шаблон для этого представления. Не волнуйтесь, это будет дальше!

:blue_square: **Примечание:** Вы можете заметить, что представление "add" для правил по-прежнему не работает, вызывая исключение `NoReverseMatch`. Это связано с тем, что мы еще не определили бэкэнды REST API, необходимые для поддержки динамических полей формы. Мы позаботимся об этом, когда разработаем функциональность REST API на девятом шаге.

<div align="center">

:arrow_left: [Шаг 4: Формы](/tutorial/step04-forms.md) | [Шаг 6: Шаблоны](/tutorial/step06-templates.md) :arrow_right:

</div>