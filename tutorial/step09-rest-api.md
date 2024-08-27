# Шаг 9: REST API

REST API обеспечивает мощную интеграцию с другими системами, которые обмениваются данными с NetBox. Он работает на [Django REST Framework](https://www.django-rest-framework.org/) (DRF), который _не_ является компонентом самого Django. В этом руководстве мы увидим, как можно расширить REST API NetBox для обслуживания нашего плагина.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step08-filter-sets`.

Наш код API будет находиться в каталоге `api/` в `netbox_access_lists/`. Давайте продолжим и создадим его, а также файл `__init__.py` сейчас:

```bash
$ cd netbox_access_lists/
$ mkdir api
$ touch api/__init__.py
```

## Создание сериализаторов моделей

Сериализаторы в некоторой степени аналогичны формам: они управляют переводом клиентских данных в объекты Python и из них, в то время как сам Django обрабатывает абстракцию базы данных. Нам нужно создать сериализатор для каждой из наших моделей. Начнем с создания `serializers.py` в каталоге `api/`.

```bash
$ edit api/serializers.py
```

В верхней части этого файла нам нужно импортировать модуль `serializers` из библиотеки `rest_framework`, а также класс `NetBoxModelSerializer` NetBox и собственные модели нашего плагина:

```python
from rest_framework import serializers

from netbox.api.serializers import NetBoxModelSerializer
from ..models import AccessList, AccessListRule
```

### Create AccessListSerializer

Сначала мы создадим сериализатор для `AccessList`, подклассифицируя `NetBoxModelSerializer`. Подобно созданию формы модели, мы создадим дочерний класс `Meta` под сериализатором, указав связанную `model` и `fields`, которые нужно включить.

```python
class AccessListSerializer(NetBoxModelSerializer):

    class Meta:
        model = AccessList
        fields = (
            'id', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

Стоит обсудить каждое из полей, которые мы назвали выше. `id` — это первичный ключ модели; его всегда следует включать в каждый сериализатор, поскольку он обеспечивает гарантированный метод уникальной идентификации объектов. Поле `display` встроено в `NetBoxModelSerializer`: это поле только для чтения, которое возвращает строковое представление объекта. Это полезно, например, для заполнения раскрывающихся списков полей формы.

Поля `name`, `default_action` и `comments` объявлены в модели `AccessList`. `tags` предоставляет доступ к менеджеру тегов объекта, а `custom_fields` включает данные его настраиваемых полей; оба они предоставляются `NetBoxModelSerializer`. Наконец, `created` и `last_updated` являются полями только для чтения, встроенными в `NetBoxModel`.

Наш сериализатор проверит модель, чтобы автоматически сгенерировать необходимые поля, однако есть одно поле, которое нам нужно добавить вручную. Каждый сериализатор должен включать поле только для чтения `url`, которое содержит URL, по которому можно получить доступ к объекту; представьте его похожим на метод `get_absolute_url()` модели. Чтобы добавить это, мы будем использовать `HyperlinkedIdentityField` DRF. Добавьте его над дочерним классом `Meta`:

```python
class AccessListSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
    view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )
```

При вызове класса поля нам нужно указать соответствующее имя представления. Обратите внимание, что это представление на самом деле еще не существует; мы создадим его немного позже.

Помните, на третьем шаге мы добавили столбец таблицы, показывающий количество правил, назначенных каждому списку доступа? Это было удобно. Давайте добавим для него поле сериализатора! Добавьте это непосредственно под полем `url`:

```python
rule_count = serializers.IntegerField(read_only=True)
```

Как и в случае со столбцом таблицы, мы будем полагаться на наше представление (которое будет определено далее), чтобы аннотировать количество правил для каждого списка доступа в базовом наборе запросов.

Наконец, нам нужно добавить `url` и `rule_count` в `Meta.fields`:

```python
class Meta:
    model = AccessList
    fields = (
        'id', 'url', 'display', 'name', 'default_action', 'comments', 'tags', 'custom_fields', 'created',
        'last_updated', 'rule_count',
    )
```

:green_circle: **Совет:** Порядок, в котором перечислены поля, определяет порядок, в котором они появляются в представлении API объекта.

### Создание AccessListRuleSerializer

Нам также нужно создать сериализатор для `AccessListRule`. Добавьте его в `serializers.py` ниже `AccessListSerializer`. Как и в случае с первым сериализатором, мы добавим класс `Meta` для определения модели и полей, а также поле `url`.

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )

    class Meta:
        model = AccessListRule
        fields = (
            'id', 'url', 'display', 'access_list', 'index', 'protocol', 'source_prefix', 'source_ports',
            'destination_prefix', 'destination_ports', 'action', 'tags', 'custom_fields', 'created',
            'last_updated',
        )
```

При ссылке на связанные объекты в сериализаторе есть еще одно соображение. По умолчанию сериализатор возвращает только первичный ключ связанного объекта; его числовой идентификатор. Это требует от клиента выполнения дополнительных запросов API для определения _любой_ другой информации о связанном объекте. Удобно автоматически включать в сериализатор некоторую информацию о связанном объекте, такую ​​как его имя и URL. Мы можем сделать это с помощью _вложенного сериализатора_.

Например, поля `source_prefix` и `destination_prefix` ссылаются на основную модель `ipam.Prefix` NetBox. Мы можем расширить `AccessListRuleSerializer`, чтобы использовать вложенный сериализатор NetBox для этой модели:

```python
from ipam.api.serializers import NestedPrefixSerializer
# ...
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail')
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

Теперь наш сериализатор будет включать сокращенное представление исходных и/или целевых префиксов для объекта. Мы должны сделать это и с полем `access_list`, однако сначала нам нужно создать вложенный сериализатор для модели `AccessList`.

### Создание вложенных сериализаторов

Начнем с импорта класса `WritableNestedSerializer` NetBox. Он будет служить базовым классом для наших вложенных сериализаторов.

```python
from netbox.api.serializers import NetBoxModelSerializer, WritableNestedSerializer
```

Затем создадим два вложенных класса сериализаторов, по одному для каждой модели нашего плагина. Каждый из них будет иметь поле `url` и дочерний класс `Meta`, как и обычные сериализаторы, однако атрибут `Meta.fields` для каждого ограничен минимальным количеством полей: `id`, `url`, `display` и дополнительный понятный человеку идентификатор. Добавьте их в `serializers.py` _выше_ обычных сериализаторов (потому что нам нужно определить `NestedAccessListSerializer`, прежде чем мы сможем ссылаться на него).

```python
class NestedAccessListSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslist-detail'
    )

    class Meta:
        model = AccessList
        fields = ('id', 'url', 'display', 'name')

class NestedAccessListRuleSerializer(WritableNestedSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )

    class Meta:
        model = AccessListRule
        fields = ('id', 'url', 'display', 'index')
```

Теперь мы можем переопределить поле `access_list` в `AccessListRuleSerializer`, чтобы использовать вложенный сериализатор:

```python
class AccessListRuleSerializer(NetBoxModelSerializer):
    url = serializers.HyperlinkedIdentityField(
    view_name='plugins-api:netbox_access_lists-api:accesslistrule-detail'
    )
    access_list = NestedAccessListSerializer()
    source_prefix = NestedPrefixSerializer()
    destination_prefix = NestedPrefixSerializer()
```

## Создание представлений

Далее нам нужно создать представления для обработки логики API. Так же, как сериализаторы примерно аналогичны формам, представления API работают аналогично представлениям пользовательского интерфейса, которые мы создали на пятом шаге. Однако, поскольку функциональность API в высокой степени стандартизирована, создание представлений существенно проще: обычно нам нужно создать только один _набор представлений_ для каждой модели. Набор представлений — это один класс, который может обрабатывать операции представления, добавления, изменения и удаления, для каждой из которых требуются выделенные представления в пользовательском интерфейсе.

Начните с создания `api/views.py` и импорта класса `NetBoxModelViewSet` NetBox, а также модулей `models` и `filtersets` нашего плагина и наших сериализаторов.

```python
from netbox.api.viewsets import NetBoxModelViewSet

from .. import filtersets, models
from .serializers import AccessListSerializer, AccessListRuleSerializer
```

Сначала мы создадим набор представлений для списков доступа, унаследовав от `NetBoxModelViewSet` и определив его атрибуты `queryset` и `serializer_class`. (Обратите внимание, что мы предварительно извлекаем назначенные теги для queryset.)

```python
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags')
    serializer_class = AccessListSerializer
```

Помните, что мы добавили поле `rule_count` в `AccessListSerializer`; давайте соответствующим образом аннотируем queryset, чтобы гарантировать заполнение этого поля (так же, как мы сделали для столбца таблицы на пятом шаге). Не забудьте импортировать служебный класс `Count` Django.

```python
from django.db.models import Count
# ...
class AccessListViewSet(NetBoxModelViewSet):
    queryset = models.AccessList.objects.prefetch_related('tags').annotate(
        rule_count=Count('rules')
    )
    serializer_class = AccessListSerializer
```

Далее мы добавим набор представлений для правил. В дополнение к `queryset` и `serializer_class` мы прикрепим набор фильтров для этой модели как `filterset_class`. Обратите внимание, что мы также предварительно извлекаем все связанные поля объектов в дополнение к тегам, чтобы повысить производительность при перечислении большого количества объектов.

```python
class AccessListRuleViewSet(NetBoxModelViewSet):
    queryset = models.AccessListRule.objects.prefetch_related(
        'access_list', 'source_prefix', 'destination_prefix', 'tags'
    )
    serializer_class = AccessListRuleSerializer
    filterset_class = filtersets.AccessListRuleFilterSet
```

## Создание URL-адресов конечных точек

Наконец, мы создадим URL-адреса конечных точек API. Это работает немного иначе, чем представления пользовательского интерфейса: вместо определения ряда путей мы создаем экземпляр _маршрутизатора_ и регистрируем каждое представление, заданное для него.

Создаем `api/urls.py` и импортируем `NetBoxRouter` NetBox и наши представления API:

```python
from netbox.api.routers import NetBoxRouter
from . import views
```

Далее мы определим `app_name`. Он будет использоваться для разрешения имен представлений API для нашего плагина.

```python
app_name = 'netbox_access_list'
```

Затем мы создаем экземпляр `NetBoxRouter` и регистрируем каждое представление с помощью нашего желаемого URL. Это конечные точки, которые будут доступны в `/api/plugins/access-lists/`.

```python
router = NetBoxRouter()
router.register('access-lists', views.AccessListViewSet)
router.register('access-list-rules', views.AccessListRuleViewSet)
```

Наконец, мы раскрываем атрибут `urls` маршрутизатора как `urlpatterns`, чтобы он был обнаружен фреймворком плагинов.

```python
urlpatterns = router.urls
```

:green_circle: **Совет:** Базовый URL для конечных точек REST API нашего плагина определяется атрибутом `base_url` класса конфигурации плагина, который мы создали на первом шаге.

Теперь, когда все наши компоненты REST API на месте, мы должны иметь возможность делать запросы API. (Обратите внимание, что вам может потребоваться сначала предоставить токен для аутентификации.) Вы можете быстро проверить, что наши конечные точки работают правильно, перейдя по адресу <http://localhost:8000/api/plugins/access-lists/> в своем браузере, войдя в NetBox. Вы должны увидеть две доступные конечные точки; нажатие на любую из них вернет список объектов.

:blue_square: **Примечание:** Если конечные точки REST API не загружаются, попробуйте перезапустить сервер разработки (`manage.py runserver`).

![REST API - Root view](/images/step09-rest-api1.png)

![REST API - Access list rules](/images/step09-rest-api2.png)

<div align="center">

:arrow_left: [Шаг 8: Наборы фильтров](/tutorial/step08-filter-sets.md) | [Шаг 10: API GraphQL](/tutorial/step10-graphql-api.md) :arrow_right:

</div>