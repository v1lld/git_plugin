# Шаг 6: Шаблоны

Шаблоны отвечают за рендеринг HTML-контента для представлений NetBox. Каждый шаблон существует как файл с комбинацией HTML и кода шаблона. Как правило, каждая модель в плагине NetBox должна иметь свой собственный шаблон. Шаблоны также могут быть созданы или настроены для других представлений, но шаблоны по умолчанию, предоставляемые NetBox, подходят в большинстве случаев.

Бэкэнд рендеринга NetBox использует [Django Template Language](https://docs.djangoproject.com/en/stable/topics/templates/) (DTL). Он сразу покажется вам очень знакомым, если вы использовали [Jinja2](https://jinja2docs.readthedocs.io/en/stable/), но имейте в виду, что между ними есть некоторые важные различия. Как правило, DTL гораздо более ограничен в типах логики, которые он может выполнять: например, непосредственное выполнение кода Python невозможно. Обязательно изучите документацию Django, прежде чем пытаться создавать какие-либо сложные шаблоны.

:blue_square: **Примечание:** Если вы пропустили предыдущий шаг, запустите `git checkout step05-views`.

## Структура файла шаблона

NetBox ищет шаблоны в каталоге `templates/` (если он существует) в корне плагина. В этом каталоге создайте подкаталог с именем плагина:

```bash
$ cd netbox_access_lists/
$ mkdir -p templates/netbox_access_lists/
```

Файлы шаблонов будут находиться в этом каталоге. Шаблоны по умолчанию предоставляются для всех общих представлений, кроме `ObjectView`, поэтому нам нужно будет создать шаблоны для наших представлений `AccessListView` и `AccessListRuleView`.

По умолчанию каждый подкласс `ObjectView` будет искать шаблон с именем связанной с ним модели. Например, `AccessListView` будет искать `accesslist.html`. Это можно переопределить, установив `template_name` в представлении, но это поведение подходит для наших целей.

## Создание шаблона AccessList

Начнем с создания файла `accesslist.html` в каталоге шаблонов плагина.

```bash
$ edit templates/netbox_access_lists/accesslist.html
```

Хотя нам нужно создать собственный шаблон, NetBox выполнил большую часть работы за нас и предоставляет общий шаблон, который мы можем легко расширить. В верхней части файла добавьте тег `extends`:

```
{% extends 'generic/object.html' %}
```

Это сообщает движку рендеринга, что сначала нужно загрузить шаблон NetBox в `generic/object.html` и заполнить только контент, который мы предоставляем в тегах `block`.

Давайте расширим блок `content` общего шаблона некоторой информацией о списке доступа.

```
{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Name</th>
              <td>{{ object.name }}</td>
            </tr>
            <tr>
              <th scope="row">Default Action</th>
              <td>{{ object.get_default_action_display }}</td>
            </tr>
            <tr>
              <th scope="row">Rules</th>
              <td>{{ object.rules.count }}</td>
            </tr>
          </table>
        </div>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
    </div>
    <div class="col col-md-6">
      {% include 'inc/panels/tags.html' %}
      {% include 'inc/panels/comments.html' %}
    </div>
  </div>
{% endblock content %}
```

Здесь мы создали строку Boostrap 5 и два столбца. В первом столбце у нас есть простая карточка для отображения имени списка доступа и действия по умолчанию, а также количества назначенных ему правил. А под ним вы увидите тег `include`, который вставляет дополнительный шаблон для отображения любых пользовательских полей, связанных с моделью. Во втором столбце мы включили еще два шаблона для отображения тегов и комментариев.

:green_circle: **Совет:** Если вы не уверены, как лучше всего построить макет страницы, есть множество примеров, на которые можно сослаться в основных шаблонах NetBox.

Давайте взглянем на наш новый шаблон! Перейдите к списку снова (по адресу <http://localhost:8000/plugins/access-lists/access-lists/>) и перейдите по ссылке к определенному списку доступа. Вы должны увидеть что-то вроде изображения ниже.

:blue_square: **Примечание:** Если NetBox жалуется, что шаблон все еще не существует, вам может потребоваться вручную перезапустить сервер разработки (`manage.py runserver`).

![Просмотр списка доступа](/images/step06-accesslist1.png)

Это хорошо, но было бы удобно также включить назначенные правила списка доступа на страницу.

### Добавить таблицу правил

Чтобы включить правила списка доступа, нам нужно предоставить дополнительные _контекстные данные_ в представлении. Откройте `views.py` и найдите класс `AccessListView`. (Это должен быть первый определенный класс.) Добавьте метод `get_extra_context()` к этому классу согласно коду ниже.

```python
class AccessListView(generic.ObjectView):
    queryset = models.AccessList.objects.all()

    def get_extra_context(self, request, instance):
        table = tables.AccessListRuleTable(instance.rules.all())
        table.configure(request)

        return {
            'rules_table': table,
        }
```

Этот метод выполняет три действия:

1. Создает экземпляр `AccessListRuleTable` с набором запросов, соответствующим всем правилам, назначенным этому списку доступа
2. Настраивает экземпляр таблицы в соответствии с текущим запросом (для соблюдения предпочтений пользователя)
3. Возвращает словарь контекстных данных, ссылающихся на экземпляр таблицы

Это делает таблицу доступной для нашего шаблона как контекстную переменную `rules_table`. Давайте добавим ее в наш шаблон.

Сначала нам нужно импортировать тег `render_table` из библиотеки `django-tables2`, чтобы мы могли отобразить таблицу как HTML. Добавьте это в верхнюю часть шаблона, сразу под строкой `{% extends 'generic/object.html' %}`:

```
{% load render_table from django_tables2 %}
```

Затем, сразу над строкой `{% endblock content %}` в конце файла, вставьте следующий код шаблона:

```
<div class="row">
<div class="col col-md-12">
<div class="card">
<h5 class="card-header">Правила</h5>
<div class="card-body table-responsive">
{% render_table rules_table %}
</div>
</div>
</div>
</div>
```

После обновления представления списка доступа в браузере вы должны увидеть таблицу правил внизу страницы.

![Просмотр списка доступа с таблицей правил](/images/step06-accesslist2.png)

## Создание шаблона AccessListRule

Говоря о правилах, не забудем о нашей модели `AccessListRule`: ей тоже нужен шаблон. Создайте `accesslistrule.html` рядом с нашим первым шаблоном:

```bash
$ edit templates/netbox_access_lists/accesslistrule.html
```

И скопируйте содержимое ниже:

```
{% extends 'generic/object.html' %}

{% block content %}
  <div class="row mb-3">
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Access List Rule</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Access List</th>
              <td>
                <a href="{{ object.access_list.get_absolute_url }}">{{ object.access_list }}</a>
              </td>
            </tr>
            <tr>
              <th scope="row">Index</th>
              <td>{{ object.index }}</td>
            </tr>
            <tr>
              <th scope="row">Description</th>
              <td>{{ object.description|placeholder }}</td>
            </tr>
          </table>
        </div>
      </div>
      {% include 'inc/panels/custom_fields.html' %}
      {% include 'inc/panels/tags.html' %}
    </div>
    <div class="col col-md-6">
      <div class="card">
        <h5 class="card-header">Details</h5>
        <div class="card-body">
          <table class="table table-hover attr-table">
            <tr>
              <th scope="row">Protocol</th>
              <td>{{ object.get_protocol_display }}</td>
            </tr>
            <tr>
              <th scope="row">Source Prefix</th>
              <td>
                {% if object.source_prefix %}
                  <a href="{{ object.source_prefix.get_absolute_url }}">{{ object.source_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Source Ports</th>
              <td>{{ object.source_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Destination Prefix</th>
              <td>
                {% if object.destination_prefix %}
                  <a href="{{ object.destination_prefix.get_absolute_url }}">{{ object.destination_prefix }}</a>
                {% else %}
                  {{ ''|placeholder }}
                {% endif %}
              </td>
            </tr>
            <tr>
              <th scope="row">Destination Ports</th>
              <td>{{ object.destination_ports|join:", "|placeholder }}</td>
            </tr>
            <tr>
              <th scope="row">Action</th>
              <td>{{ object.get_action_display }}</td>
            </tr>
          </table>
        </div>
      </div>
    </div>
  </div>
{% endblock content %}
```

На этом этапе вы, вероятно, сможете сказать, что делает большая часть кода шаблона выше, но вот несколько деталей, которые стоит упомянуть:

* URL для родительского списка доступа правила извлекается путем вызова `object.access_list.get_absolute_url()` (метод, который мы добавили на пятом шаге), _без_ скобок (отличие DTL). Этот метод также используется для связанных префиксов.
* Фильтр `placeholder` NetBox применяется к описанию правила. (Это отображает &mdash; для пустых полей.)
* Атрибуты `protocol` и `action` отображаются путем вызова, например, `object.get_protocol_display()` (снова без скобок). Это [соглашение Django](https://docs.djangoproject.com/en/stable/ref/models/instances/#extra-instance-methods) для статических полей выбора, чтобы возвращать понятную человеку метку, а не необработанное значение.

![Просмотр правила списка доступа](/images/step06-accesslistrule.png)

Не стесняйтесь экспериментировать с различными макетами и содержимым, прежде чем переходить к следующему шагу.

<div align="center">

:arrow_left: [Шаг 5: Виды](/tutorial/step05-views.md) | [Шаг 7: Навигация](/tutorial/step07-navigation.md) :arrow_right:

</div>