# LiveConfigs

LiveConfigs помогает настраивать приложение "на лету" через django-админку без перевыкатки и перезапуска. 

Сама конфигурация хранится в БД, но для разработчика скрыта за удобным программным интерфейсом.

## Быстрый пример

Создаем настройки

```python
class MyConfig(liveconfigs.BaseConfig):
   IS_FEATURE_ENABLED: bool = False
   NEW_FEATURE_VALUE: float = 20.4
```

Используем их

```python
my_config = config.MyConfig().get()
if my_config.IS_FEATURE_ENABLED:
   print('feature is enabled and feature value is', my_config.NEW_FEATURE_VALUE)
```

Меняем и отслеживаем изменения через админку!

![Экран редактирования конфига](https://github.com/liveconfigs/django-liveconfigs/blob/main/images/change_config.jpg?raw=true)

![История изменений конфигов](https://github.com/liveconfigs/django-liveconfigs/blob/main/images/history.jpg?raw=true)


## Термины
+ __настройка__, __конфиг__ - одно типизированное именованное значение
+ __класс конфига__, __группа настроек__ - настройки, объединенные в один класс python
+ __топик__ - общая "тема" для группы настроек, по которой можно производить поиск настроек в админке.
У разных групп настроек в принципе может быть один и тот же топик
+ __теги__ - ключевые слова для быстрого поиска настроек. Одна настройка может иметь несколько тегов
## Со стороны разработчика

Для разработчика настройки выглядят как статические члены-константы классов python.
Их можно быстро добавить, убрать, их просто использовать (пара строк кода). Их понимает mytype и среды разработки.

За создание новых конфигов отвечает разработчик.
Значения по-умолчанию хранятся в коде.
В БД новые конфиги попадают при выкатке или при первом использовании.


## Со стороны администратора 
Значения задаются через админку django. 
Для удобного администрирования конфиги поддерживают
+ __теги__ и __топики__ для группировки и поиска
+ документацию
+ автоматическую загрузку новых конфигов в бд
+ валидацию при изменении (типы и значения)
+ сохранение даты последних изменений и чтений конфигов
+ импорт и экспорт (удобно для тестирования системы)

![Экран поиска и фильтрации конфигов](https://github.com/liveconfigs/django-liveconfigs/blob/main/images/filter_config.jpg?raw=true)

## Установка
### 1. Выполните установку пакета через pip, poetry или другим способом
```
 pip install django-liveconfigs
```

### 2. В настройках проекта:
- **ВКЛЮЧИТЕ `liveconfigs` В РАЗДЕЛЕ ПРИЛОЖЕНИЙ, ДОБАВИВ ДВЕ СТРОКИ:**
```python
INSTALLED_APPS = [
    ...
    "import_export",
    "liveconfigs",
]
```

- **ДОБАВЬТЕ НАСТРОЙКИ**

> Если вас устраивают значения по умолчанию (они использованы ниже), то настройку можно не описывать.

```python
LC_BACKGROUND_SAVE = False
```
Этот параметр отвечает за режим записи вспомогательных данных (время изменения, время последнего доступа к конкретному конфигу) в базу данных.

`True` - сохранение происходит в фоне через сигнал (например, в Celery)

`False` - сохранение происходит в основном потоке программы

> При асинхронной заморозке сохранение всегда происходит в "основном потоке" (синхронная или асинхронная функция)

```python
LC_CACHE_TTL = 1
```
Время (в секундах) в течение которого кешированное значение конфига валидно

```python
LC_LAST_READ_UPDATE_TTL = 60*60*24
```
Минимальный период (в секундах) между обновлениями даты чтения конфига в БД.

В примере: информация о дате последнего чтения конфига будет обновляться не чаще 1 раза в сутки.

```python
LC_ENABLE_PRETTY_INPUT = False
```
Этот параметр отвечает за включение "типизированных" (основанных на типе конфига) полей ввода при редактировании конфига в админке.

> Не работает при использовании Union в типе конфига

```python
LC_MAX_VISUAL_VALUE_LENGTH = 0
```
Максимальная длина значения конфига (в текстовом представлении) при которой значение в списке выводится целиком. 
При бОльшей длине визуал значения будет усечен ("Длинная строка" -> "Длин ... рока")
> 0 - настройка отключена

```python
LC_MAX_VISUAL_VALUE_LENGTH = 50
```
Максимальная длина текста в значении конфига при которой отображать поле редактирования конфига как `textinput` (однострочный редактор)

При длине текста в значении конфига большей этого значения - отображать поле редактирования конфига как `textarea` (многострочный редактор)

> Используется для конфигов с типом `str`

## Использование

### 1. Создайте файл с конфигами в удобном месте проекта

Например, файл `config.py` в директории `config`, созданной на верхнем уровне проекта.

### 2. Отредактируйте файл с конфигами

> Упрощенный вариант для `LC_BACKGROUND_SAVE = False`:

```python
from liveconfigs import models
from liveconfigs.validators import greater_than # используйте нужные вам валидаторы
from enum import Enum

# isort: off
# config_row_update_signal_handler begin
from django.conf import settings
from django.dispatch import receiver
from liveconfigs.signals import config_row_update_signal
from liveconfigs.tasks import config_row_update_or_create
# isort: on


@receiver(config_row_update_signal, dispatch_uid="config_row_update_signal")
def config_row_update_signal_handler(sender, config_name, update_fields, **kwargs):
    config_row_update_or_create(config_name, update_fields)

# config_row_update_signal_handler end
```

> Полный вариант для `LC_BACKGROUND_SAVE = True`:

```python
from liveconfigs import models
from liveconfigs.validators import greater_than # используйте нужные вам валидаторы
from enum import Enum

# isort: off
# config_row_update_signal_handler begin
from django.conf import settings
from django.dispatch import receiver
from liveconfigs.signals import config_row_update_signal
from liveconfigs.tasks import config_row_update_or_create

# FIXME: Импорт приложения Celery из вашего проекта (если используете Celery)
# FIXME: Вам нужно изменить этот код, если вы используете не Celery
from celery_app import app
# isort: on

# Пример для Celery
# Реальное сохранение данных выполняет функция config_row_update_or_create
# Реализуйте отложенное сохранение удобным для вас методом
# Для Celery зарегистрируйте эту задачу в CELERY_TASK_ROUTES
# FIXME: Вам нужно изменить этот код, если вы используете не Celery
@app.task(max_retries=1, soft_time_limit=1200, time_limit=1500)
def config_row_update_or_create_proxy(config_name: str, update_fields: dict):
    config_row_update_or_create(config_name, update_fields)


@receiver(config_row_update_signal, dispatch_uid="config_row_update_signal")
def config_row_update_signal_handler(sender, config_name, update_fields, **kwargs):
    LIVECONFIGS_SYNCWRITE = getattr(settings, 'LIVECONFIGS_SYNCWRITE', 'True')
    LC_BACKGROUND_SAVE = getattr(settings, 'LC_BACKGROUND_SAVE', 'False')
    background_save = LC_BACKGROUND_SAVE if hasattr(settings, 'LC_BACKGROUND_SAVE') else not LIVECONFIGS_SYNCWRITE

    # Пример для Celery
    # При настройках для фонового сохранения функция будет вызвана через delay
    # FIXME: Вам нужно изменить этот код, если вы используете не Celery
    if background_save:
        config_row_update_or_create_proxy_func = config_row_update_or_create_proxy.delay
    # При настройках для синхронного сохранения функция будет вызвана напрямую
    else:
        config_row_update_or_create_proxy_func = config_row_update_or_create_proxy

    config_row_update_or_create_proxy_func(config_name, update_fields)

# config_row_update_signal_handler end
```

### 3. Создайте нужные конфиги в файле
```python
# тут перечислены возможные теги для настроек из вашей предметной области
class ConfigTags(str, Enum):
    front = "Настройки для фронта"
    features = "Фичи"
    basic = "Основные"
    other = "Прочее"

# тут описана сама настройка и ее мета-данные
class FirstExample(models.BaseConfig):
    __topic__ = 'Основные настройки'  # короткое описание группы настроек
    MY_FIRST_CONFIG: int = 40
    # следующие строчки необязательны
    MY_FIRST_CONFIG_DESCRIPTION = "Какая-то моя настройка"
    MY_FIRST_CONFIG_TAGS = [ConfigTags.basic, ConfigTags.other]
    MY_FIRST_CONFIG_VALIDATORS = [greater_than(5)]
    
    # вторая настройка, без метаданных
    SECOND_ONE: bool = False  
```

### 4. Используйте конфиги в коде

```python
from config.config import FirstExample
...
# в синхронном коде
first_example = FirstExample().get()
# в асинхронном коде
first_example = await FirstExample().aget()
...
if first_example.MY_FIRST_CONFIG > 20:
    print("Hello there!")
```

## Просмотр и редактирование
Редактировать значения конфигов можно по адресу
 http://YOUR_HOST/admin/liveconfigs/configrow/

При установке значений проверяется тип нового значения, а также вызываются
дополнительные валидаторы

## Автоматическая загрузка новых конфигов в БД
При первом обращении к настройке приложение проверяет, есть ли запись о ней в БД. Если ее нет, то конфиг записывается в БД со значением по-умолчанию.

Если по какой-то причине вы не хотите ждать, то залить все новые конфиги в БД можно и на старте сервиса, добавив в ваш скрипт запуска вызов команды `load_config`:

```sh
    # какие-то старые команды
    python /app/manage.py migrate --noinput
    python /app/manage.py collectstatic --noinput

    #  НОВАЯ СТРОЧКА - загрузка конфигов в бд
    python /app/manage.py load_config

    #  сам запуск сервиса - может быть и так
    python /app/manage.py runserver_plus 0.0.0.0:8080 --insecure
```

## Даты последнего изменения и чтения
В БД у каждой настройки есть два дополнительных поля - даты последнего чтения и записи. Они помогают определить в живой системе, нужны ли все еще какие-то настройки или пора уже от них избавиться.

## Ограничения
- Для Django < 4 поддерживается только Postgres
- Для Django < 4.2 асинхронная заморозка может не работать
- Фоновое сохранение данных о последнем чтении конфига не работает в асинхронной заморозке

## Остались вопросы?
+ Посмотрите примеры использования конфигов: https://github.com/liveconfigs/django-liveconfigs-example/

+ Примеры использования валидаторов : https://github.com/liveconfigs/django-liveconfigs-example/

+ Примеры валидаторов : `validators.py` в https://github.com/liveconfigs/django-liveconfigs/

+ Напишите нам! 
