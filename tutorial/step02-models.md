# Шаг 2: Модели

На этом шаге мы определим несколько моделей Django для хранения данных нашего плагина. Модель — это класс Python, который представляет таблицу в базовой базе данных PostgreSQL; каждый экземпляр модели соответствует строке в таблице. Мы используем модели вместо чистого SQL, поскольку взаимодействие с объектами Python гораздо удобнее и гибче.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, выполните `git checkout step01-initial-setup`.

## Создание моделей

Сначала перейдите `cd` в каталог `netbox_access_lists` и создайте файл с именем `models.py`. Именно здесь будут определены наши классы моделей.

```bash
$ cd netbox_access_lists
$ edit models.py
```

В верхней части файла импортируйте библиотеку `models` Django и класс `NetBoxModel` NetBox. Последний будет служить базовым классом для моделей нашего плагина. Мы также импортируем PostgreSQL `ArrayField`; подробнее об этом чуть позже.

```python
из django.contrib.postgres.fields импорт ArrayField
из django.db импорт models
из netbox.models импорт NetBoxModel
```

Мы создадим две модели:

* `AccessList`: это будет представлять список доступа с именем и одним или несколькими назначенными ему правилами.
* `AccessListRule`: это будет отдельное правило с IP-адресами источника/назначения, номерами портов и т. д., назначенными списку доступа.

### AccessList

Нам нужно будет определить несколько полей для нашей модели. Каждая модель автоматически получает числовое поле первичного ключа (`id`), поэтому нам не нужно беспокоиться об этом, но нам нужно определить поля для имени ACL, действия по умолчанию и необязательных комментариев.

```python
class AccessList(NetBoxModel):
name = models.CharField(
max_length=100
)
default_action = models.CharField(
max_length=30
)
comments = models.TextField(
blank=True
)
```

По умолчанию экземпляры моделей упорядочиваются по их первичным ключам, но было бы разумнее упорядочивать списки доступа по имени. Мы можем сделать это, создав дочерний класс `Meta` и определив переменную `ordering`. (Обязательно создайте класс `Meta` *внутри* `AccessList`, а не после него.)

```python
class Meta:
ordering = ('name',)
```

Наконец, мы добавим метод `__str__()` для управления приведением экземпляра к строке. Он будет возвращать значение поля `name` экземпляра. (Опять же, обязательно создайте этот метод *внутри* класса `AccessList`.)

```python
def __str__(self):
return self.name
```

### AccessListRule

Наша вторая модель будет содержать отдельные правила, назначенные каждому списку доступа. Эта модель будет немного сложнее. Нам нужно будет определить поля для всего следующего:

* Родительский список доступа (внешний ключ к экземпляру `AccessList`)
* Индекс (порядок правила в списке)
* Протокол
* Исходный префикс
* Исходный порт(ы)
* Префикс назначения
* Целевой порт(ы)
* Действие (разрешить, разрешить или отклонить)
* Описание (необязательно)

Начнем с определения поля `ForeignKey`, указывающего на модель `AccessList`.

```python
class AccessListRule(NetBoxModel):
access_list = models.ForeignKey(
to=AccessList,
on_delete=models.CASCADE,
related_name='rules'
)
```

Мы передаем три ключевых аргумента в поле:

* `to` ссылается на связанный класс модели
* `on_delete` сообщает Django, какое действие следует предпринять, если связанный объект удален. `CASCADE` автоматически удалит все правила, назначенные удаленному списку доступа.
* `related_name` определяет атрибут обратной связи, добавляемой к связанному классу. Правила, назначенные экземпляру `AccessList`, можно ссылаться как `accesslist.rules.all()`.

Далее мы добавим поле `index` для хранения номера правила (позиции) в списке доступа. Мы будем использовать `PositiveIntegerField`, поскольку поддерживаются только положительные числа.

```python
index = models.PositiveIntegerField()
```

Далее следует поле протокола. Оно будет хранить имя протокола, например TCP или UDP. Обратите внимание, что мы устанавливаем `blank=True`, поскольку при создании правила не требуется указывать конкретный протокол.

```python
protocol = models.CharField(
max_length=30,
blank=True
)
```

Далее нам нужно определить исходный префикс. Мы собираемся использовать поле внешнего ключа для ссылки на экземпляр модели `Prefix` NetBox в ее приложении `ipam`. Вместо импорта класса модели мы можем ссылаться на него по имени. И поскольку мы хотим, чтобы это было _необязательное_ поле, мы также установим `blank=True` и `null=True`.

```python
source_prefix = models.ForeignKey(
to='ipam.Prefix',
on_delete=models.PROTECT,
related_name='+',
blank=True,
null=True
)
```

:green_circle: **Совет:** В то время как `CASCADE` автоматически удаляет дочерние объекты, `PROTECT` предотвращает удаление родительского параметра, если существуют какие-либо дочерние объекты.

Обратите внимание, что выше мы определили `related_name='+'`. Это говорит Django не создавать обратную связь из модели `Prefix` в модель `AccessListRule`, потому что это было бы не очень полезно.

Нам также нужно добавить поле для номера исходного порта. Для этого мы могли бы использовать целочисленное поле, однако это ограничило бы нас определением одного исходного порта на правило. Вместо этого мы можем добавить `ArrayField` для хранения списка значений `PositiveIntegerField`. Как и `source_prefix`, это также будет необязательным полем, поэтому мы также добавляем `blank=True` и `null=True`.

```python
source_ports = ArrayField(
base_field=models.PositiveIntegerField(),
blank=True,
null=True
)
```

Давайте продолжим и добавим поля префикса назначения и порта. По сути, это дубликаты наших исходных полей.

```python
destination_prefix = models.ForeignKey(
to='ipam.Prefix',
on_delete=models.PROTECT,
related_name='+',
blank=True,
null=True
)
destination_ports = ArrayField(
base_field=models.PositiveIntegerField(),
blank=True,
null=True
)
```

Наконец, мы добавим поля для действия и описания правила. Действие обязательно, а описание — нет.

```python
action = models.CharField(
max_length=30
)
description = models.CharField(
max_length=500,
blank=True
)
```

После того, как наши поля будут удалены, этой модели также понадобится класс `Meta` для определения порядка в базе данных и для обеспечения того, чтобы каждое правило имело уникальный номер индекса в родительском списке доступа.

```python
class Meta:
ordering = ('access_list', 'index')
unique_together = ('access_list', 'index')
```

Наконец, мы добавим метод `__str__()` для отображения родительского списка доступа и индексного номера при отображении экземпляра `AccessListRule` в виде строки:

```python
def __str__(self):
return f'{self.access_list}: Rule {self.index}'
```

## Определение вариантов полей

Оглядываясь назад на наши модели, мы видим несколько полей, которые выиграли бы от наличия предопределенных вариантов, из которых пользователь может выбирать при создании или изменении экземпляра. В частности, мы ожидаем, что поле `action` правила будет иметь только одно из трех значений:

* Разрешить
* Запретить
* Отклонить

Мы можем определить `ChoiceSet` для хранения этих предопределенных значений для пользователя, чтобы избежать хлопот с ручным вводом имени нужного действия каждый раз. Вернитесь в начало `models.py`, импортируйте класс `ChoiceSet` NetBox:

```python
from utilities.choices import ChoiceSet
```

Затем под операторами импорта, но над определениями моделей создайте дочерний класс с именем `ActionChoices`:

```python
class ActionChoices(ChoiceSet):
key = 'AccessListRule.action'

CHOICES = [
('permit', 'Permit', 'green'),
('deny', 'Deny', 'red'),
('reject', 'Reject (Reset)', 'orange'),
]
```

Атрибут `CHOICES` должен быть итерируемым из двух- или трехзначных кортежей, каждый из которых определяет следующее:

* Необработанное значение, которое будет сохранено в базе данных
* Удобное для человека строка для отображения
* Цвет для отображения в пользовательском интерфейсе (необязательно, см. [доступные цвета](https://docs.netbox.dev/en/stable/configuration/data-validation/#field_choices))

Кроме того, мы добавили атрибут `key`: он позволит администратору NetBox заменить или расширить выбор плагина по умолчанию с помощью параметра конфигурации NetBox [`FIELD_CHOICES`](https://netbox.readthedocs.io/en/stable/configuration/optional-settings/#field_choices).

Теперь мы можем ссылаться на это как на набор допустимых вариантов в полях модели `default_action` и `action`, передавая его как аргумент ключевого слова `choices`.

```python
# AccessList
default_action = models.CharField(
max_length=30,
choices=ActionChoices
)

# AccessListRule
action = models.CharField(
max_length=30,
choices=ActionChoices
)
```

Давайте также создадим набор вариантов для поля `protocol` правила. Добавьте это под классом `ActionChoices`:

```python
class ProtocolChoices(ChoiceSet):

CHOICES = [
('tcp', 'TCP', 'blue'),
('udp', 'UDP', 'orange'),
('icmp', 'ICMP', 'purple'),
]
```

Затем добавьте аргумент ключевого слова `choices` в поле `protocol`:

```python
# AccessListRule
protocol = models.CharField(
max_length=30,
choices=ProtocolChoices,
blank=True
)
```


## Создание миграций схемы

Теперь, когда у нас определены модели, нам нужно сгенерировать схему для базы данных PostgreSQL. Хотя таблицы и ограничения можно создавать вручную, гораздо проще использовать [функцию миграции](https://docs.djangoproject.com/en/4.0/topics/migrations/) Django. Она проверит наши классы модели и автоматически сгенерирует необходимые файлы миграции. Это двухэтапный процесс: сначала мы генерируем файл миграции с помощью команды управления `makemigrations`, затем запускаем `migrate`, чтобы применить его к рабочей базе данных.

:warning: **Предупреждение:** Перед продолжением проверьте, что вы установили `DEVELOPER=True` в файле `configuration.py` NetBox. Это необходимо для отключения защиты, призванной не допустить ошибочного создания новых миграций.

### Генерация файлов миграции

Перейдите в корневой каталог установки NetBox, чтобы запустить `manage.py`. Сначала мы запустим `makemigrations` с аргументом `--dry-run` в качестве проверки работоспособности. Это сообщит об обнаруженных изменениях, но не сгенерирует никаких файлов миграции.

```bash
$ python netbox/manage.py makemigrations netbox_access_lists --dry-run
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    - Create model AccessList
    - Create model AccessListRule
```

Мы должны увидеть план создания первого файла миграции нашего плагина, `0001_initial.py`, с двумя моделями, которые мы определили в `models.py`. (Если на этом этапе вы столкнулись с ошибкой или не видите вывод выше, **остановитесь здесь** и проверьте свою работу.) Если все выглядит хорошо, продолжайте создавать файл миграции (опуская аргумент `--dry-run`):

```bash
$ python netbox/manage.py makemigrations netbox_access_lists
Migrations for 'netbox_access_lists':
  ~/netbox-plugin-demo/netbox_access_lists/migrations/0001_initial.py
    - Create model AccessList
    - Create model AccessListRule
```

Вернувшись в рабочую область плагина, вы должны увидеть каталог `migrations` с двумя файлами: `__init__.py` и `0001_initial.py`.

```bash
$ tree
.
├── __init__.py
├── миграции
│   ├── 0001_initial.py
│   ├── __init__.py
...
```

### Применить миграции

Наконец, мы можем применить файл миграции с помощью команды управления `migrate`:

```bash
$ python netbox/manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, circuits, contenttypes, dcim, django_rq, extras, ipam, netbox_access_lists, sessions, social_django, taggit, tenancy, users, virtualization, wireless
Running migrations:
  Applying netbox_access_lists.0001_initial... OK
```

Если вам интересно, вы можете проверить недавно созданные таблицы базы данных, используя команду управления `dbshell` для входа в оболочку PostgreSQL:

```bash
$ python netbox/manage.py dbshell
psql (10.19 (Ubuntu 10.19-0ubuntu0.18.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

netbox=> \d netbox_access_lists_accesslist
                                          Table "public.netbox_access_lists_accesslist"
      Column       |           Type           | Collation | Nullable |                          Default                           
-------------------+--------------------------+-----------+----------+------------------------------------------------------------
 id                | bigint                   |           | not null | nextval('netbox_access_lists_accesslist_id_seq'::regclass)
 created           | timestamp with time zone |           |          | 
 last_updated      | timestamp with time zone |           |          | 
 custom_field_data | jsonb                    |           | not null | 
 name              | character varying(100)   |           | not null | 
 default_action    | character varying(30)    |           | not null | 
 comments          | text                     |           | not null | 
Indexes:
    "netbox_access_lists_accesslist_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "netbox_access_lists_accesslistrule" CONSTRAINT "netbox_access_lists__access_list_id_6c1b0317_fk_netbox_ac" FOREIGN KEY (access_list_id) REFERENCES netbox_access_lists_accesslist(id) DEFERRABLE INITIALLY DEFERRED
```

Введите `\q`, чтобы выйти из `dbshell`.

## Создайте несколько объектов

Теперь, когда у нас установлены модели, давайте попробуем создать несколько объектов. Сначала войдите в оболочку NetBox. Это интерактивный интерфейс командной строки Python, который позволяет нам напрямую взаимодействовать с объектами NetBox и другими ресурсами.

```bash
$ python netbox/manage.py nbshell
from netbox### NetBox interactive shell
### Python 3.8.12 | Django 4.0.3 | NetBox 3.2.0
### lsmodels() will show available models. Use help(<model>) for more info.
>>>
```

Давайте создадим и сохраним список доступа:

```python
>>> from netbox_access_lists.models import *
>>> acl = AccessList(name='MyACL1', default_action='deny')
>>> acl
<AccessList: MyACL1>
>>> acl.save()
```

Далее мы создадим несколько префиксов для ссылок в правилах:

```python
>>> prefix1 = Prefix(prefix='192.168.1.0/24')
>>> prefix1.save()
>>> prefix2 = Prefix(prefix='192.168.2.0/24')
>>> prefix2.save()
```

И наконец мы создадим пару правил для нашего списка доступа:

```python
>>> AccessListRule(
... access_list=acl,
... index=10,
... protocol='tcp',
... destination_prefix=prefix1,
... destination_ports=[80, 443],
... action='permit',
... description='Веб-трафик'
... ).save()
>>> AccessListRule(
... access_list=acl,
... index=20,
... protocol='udp',
... destination_prefix=prefix2,
... destination_ports=[53],
... action='permit',
... description='DNS'
... ).save()
>>> acl.rules.all()
<RestrictedQuerySet [<AccessListRule: MyACL1: Rule 10>, <AccessListRule: MyACL1: Rule 20>]>
```

Отлично! Теперь мы можем создавать списки доступа и правила в базе данных. Следующие несколько шагов будут направлены на раскрытие этой функциональности в пользовательском интерфейсе NetBox.

<div align="center">

:arrow_left: [Шаг 1: Начальная настройка](/tutorial/step01-initial-setup.md) | [Шаг 3: Таблицы](/tutorial/step03-tables.md) :arrow_right:

</div>