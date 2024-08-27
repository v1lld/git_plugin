# NetBox Plugin Development Tutorial

Это руководство призвано продемонстрировать процесс разработки пользовательского плагина для NetBox v3.2 или более поздней версии. Выполняя каждый из предписанных шагов, читатель создаст с нуля простой плагин для управления списками доступа в NetBox, используя все основные компоненты фреймворка плагинов NetBox.

Завершенная копия демонстрационного плагина, созданного в этом руководстве, доступна в репозитории [`netbox-plugin-demo`](https://github.com/v1lld/netbox_access_lists) для справки. Для вашего удобства завершенный код, соответствующий каждому шагу в руководстве, существует в виде именованной ветви в демонстрационном репозитории. Например, если вы хотите начать заново на шаге 5, просто проверьте ветку `step04-forms`.

### Предварительные условия

Перед тем, как попытаться создать плагин, пожалуйста, оцените свои личные способности. Авторы плагинов должны иметь достаточные навыки в следующих областях:

* Программирование на Python
* Фреймворк [Django](https://www.djangoproject.com/)
* Основы REST API
* Установка, настройка и использование NetBox

### Содержание

* [Шаг 1: Начальная настройка](/tutorial/step01-initial-setup.md) :arrow_left: Начните здесь!
* [Шаг 2: Модели](/tutorial/step02-models.md)
* [Шаг 3: Таблицы](/tutorial/step03-tables.md)
* [Шаг 4: Формы](/tutorial/step04-forms.md)
* [Шаг 5: Представления](/tutorial/step05-views.md)
* [Шаг 6: Шаблоны](/tutorial/step06-templates.md)
* [Шаг 7: Навигация](/tutorial/step07-navigation.md)
* [Шаг 8: Наборы фильтров](/tutorial/step08-filter-sets.md)
* [Шаг 9: REST API](/tutorial/step09-rest-api.md)
* [Шаг 10: GraphQL API](/tutorial/step10-graphql-api.md)
* [Шаг 11: Поиск](/tutorial/step11-search.md)

### Ссылка

* [Документация по разработке плагинов NetBox](https://netbox.readthedocs.io/en/stable/plugins/development/)
* [Программа сертификации плагинов NetBox Labs](https://github.com/netbox-community/netbox/wiki/Plugin-Certification-Program)

