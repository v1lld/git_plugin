# Шаг 1: Начальная настройка

Прежде чем мы начнем работу над нашим плагином, мы должны сначала убедиться, что у нас есть подходящая среда разработки.

## Настройка среды разработки

### Установка NetBox

Для разработки плагина требуется локальная установка NetBox. Если у вас еще не установлен NetBox, ознакомьтесь с [инструкциями по установке](https://netbox.readthedocs.io/en/stable/installation/).

Обязательно включите отладку в конфигурации NetBox, установив `DEBUG = True`. Это гарантирует, что статические ресурсы могут обслуживаться сервером разработки и возвращать полные трассировки при возникновении ошибки сервера.

:green_circle: **Совет:** Если эта установка будет использоваться только для разработки, обычно необходимо выполнить только до третьего шага в руководстве по установке, завершаясь успешным запуском сервера разработки NetBox (`manage.py runserver`).

:warning: **Предупреждение:** Для этого руководства требуется NetBox v3.2 или более поздняя версия. Попытка использовать более раннюю версию NetBox не сработает.

### Клонирование репозитория git

Далее мы клонируем демонстрационный репозиторий git с GitHub. Сначала `cd` в предпочитаемое вами место (ваш домашний каталог, вероятно, подойдет), затем клонируем репозиторий с помощью `git clone`. Мы проверяем ветку `init`, которая предоставит нам пустое рабочее пространство для начала.

```bash
$ git clone --branch step00-empty https://github.com/netbox-community/netbox-plugin-demo
Cloning into 'netbox-plugin-demo'...
remote: Enumerating objects: 58, done.
remote: Counting objects: 100% (58/58), done.
remote: Compressing objects: 100% (42/42), done.
remote: Total 58 (delta 12), reused 58 (delta 12), pack-reused 0
Unpacking objects: 100% (58/58), done.
```

:blue_square: **Примечание:** Клонирование демонстрационного репозитория не является строго обязательным, но это позволит вам удобно проверять снимки кода по мере прохождения уроков и преодолевать любые сбои.

## Конфигурация плагина

### Создание `__init__.py`

Класс `PluginConfig` содержит всю информацию, которую необходимо знать о нашем плагине для его установки. Сначала мы создадим подкаталог для хранения кода Python нашего плагина, а также файл `__init__.py` для хранения определения `PluginConfig`.

```bash
$ mkdir netbox_access_lists
$ touch netbox_access_lists/__init__.py
```

Далее откройте `__init__.py` в текстовом редакторе по вашему выбору и импортируйте класс `PluginConfig` из NetBox в верхней части файла.

```python
from netbox.plugins import PluginConfig
```

### Создайте класс PluginConfig

Мы создадим новый класс с именем `NetBoxAccessListsConfig`, создав подкласс `PluginConfig`. Это определит все необходимые параметры, которые управляют конфигурацией нашего плагина после установки. Существует [множество необязательных атрибутов](https://netbox.readthedocs.io/en/stable/plugins/development/#pluginconfig-attributes), которые можно задать здесь, но сейчас нам нужно определить только несколько.

```python
class NetBoxAccessListsConfig(PluginConfig):
name = 'netbox_access_lists'
verbose_name = ' NetBox Access Lists'
description = 'Manage simple ACLs in NetBox'
version = '0.1'
base_url = 'access-lists'
```

Этого будет достаточно для установки нашего плагина в NetBox позже. Наконец, нам нужно предоставить этот класс как `config`, чтобы NetBox его обнаружил. Добавьте эту строку в конец файла:

```python
config = NetBoxAccessListsConfig
```

## Создайте файл README

Считается хорошей практикой всегда включать файл `README` в любой публикуемый вами код. Это краткий документ, который объясняет цель вашего проекта, как его установить/запустить, где найти помощь и т. д. Поскольку это всего лишь обучающее упражнение, нам нечего сказать о нашем плагине, но все равно создайте файл.

Вернитесь в корень проекта (на один уровень выше от `__init__.py`), создайте файл с именем `README.md` и введите следующее содержимое:

```markdown
## netbox-access-lists

Manage simple access control lists in NetBox
```

:green_circle: **Совет:** Вы заметите, что мы дали нашему файлу `README` расширение `md`. Это сообщает инструментам, которые его поддерживают, о необходимости отображать файл как Markdown для лучшей читаемости.

## Установка плагина

### Создание `setup.py`

Чтобы включить установку нашего плагина в виртуальную среду, которую мы создали выше, мы создадим простой скрипт установки Python. В корневом каталоге проекта создайте файл с именем `setup.py` и введите код ниже.

```python
from setuptools import find_packages, setup

setup(
name='netbox-access-lists',
version='0.1',
description='An example NetBox plugin',
install_requires=[],
packages=find_packages(),
include_package_data=True,
zip_safe=False,
)
```
:warning: **Предупреждение:** Обязательно создайте `setup.py` в корне проекта, а не в каталоге `netbox_access_lists`.

Этот файл вызовет функцию `setup()`, предоставляемую библиотекой Python [`setuptools`](https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/), для установки нашего кода. Существует множество дополнительных аргументов, которые можно передать, но для нашего примера этого достаточно.

:green_circle: **Совет:** Существуют альтернативные методы установки кода Python, которые работают так же хорошо; смело используйте свой предпочтительный подход. Просто помните, что это руководство предполагает использование `setuptools`, и вносите соответствующие изменения.

### Активация виртуальной среды

Чтобы убедиться, что наш плагин доступен для установки NetBox, нам сначала нужно активировать [виртуальную среду] Python (https://docs.python.org/3/library/venv.html), которая была создана при установке NetBox. Для этого определите путь к виртуальной среде (это будет `/opt/netbox/venv/`, если вы используете значения по умолчанию из документации) и активируйте ее:

```bash
$ source /opt/netbox/venv/bin/activate
```

### Запустите `setup.py`

Теперь мы можем установить наш плагин, запустив `setup.py`. Сначала убедитесь, что виртуальная среда все еще активна, затем выполните следующую команду из корня проекта. Аргумент `develop` сообщает `setuptools` о необходимости создать ссылку на наш локальный путь разработки вместо копирования файлов в виртуальную среду. Это позволяет избежать необходимости переустанавливать плагин каждый раз, когда мы вносим изменения.

```bash
$ python3 setup.py develop
running develop
running egg_info
creating netbox_access_lists.egg-info
writing manifest file 'netbox_access_lists.egg-info/SOURCES.txt'
writing manifest file 'netbox_access_lists.egg-info/SOURCES.txt'
running build_ext
```

Если вы видите это:

```bash
django.core.exceptions.ImproperlyConfigured: Unable to import plugin netbox_access_list: Module not found. Check that the plugin module has been installed within the correct Python environment.
```

Используйте 

```bash
sudo /opt/netbox/venv/bin/python3 -m pip install -e [path]
```
путь, по которому вы извлекли плагин, вместо [path].

### Настройка NetBox

Наконец, нам нужно настроить NetBox для включения нашего нового плагина. В пути установки NetBox откройте `netbox/netbox/configuration.py` и найдите параметр `PLUGINS`; это должен быть пустой список. (Если он еще не определен, создайте его.) Добавьте имя нашего плагина в этот список:

```python
# configuration.py
PLUGINS = [
'netbox_access_lists',
]
```

Сохраните файл и запустите сервер разработки NetBox (если он еще не запущен):

```bash
$ python netbox/manage.py runserver
```

Вы должны увидеть, что сервер разработки успешно запустился. Откройте NetBox в новом окне браузера, войдите в систему как суперпользователь и перейдите в интерфейс администратора. В разделе **Система > Установленные плагины** вы должны увидеть наш плагин в списке.

![Интерфейс администратора Django: список плагинов](/images/step01-django-admin-plugins.png)

:green_circle: **Совет:** Вы можете проверить свою работу в конце каждого шага в руководстве, запустив `git diff` для соответствующей ветки. Например, в конце первого шага выполните `git diff remotes/origin/step01-initial-setup`, чтобы сравнить свою работу с завершенным шагом. Это поможет выявить любые задачи, которые вы могли пропустить.

Это завершает нашу первоначальную настройку. Теперь самое интересное!

<div align="center">

[Шаг 2: Модели](/tutorial/step02-models.md) :arrow_right:

</div>
