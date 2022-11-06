# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Деплой сайта в кластер Minikube

1. Установите Minikube, Kubectl и Helm согласно документации. Создайте кластер.
2. Загрузите образ django_app, созданный docker-compose, в кластер:  
```shell-session
$ minikube image load django_app 
```
3. Перейдите в папку /kubernetes и создайте  файл django.env с переменными окружения, описанными выше. 
Для ALLOWED_HOSTS укажите star-burger.test.
Формат DATABASE_URL "postgres://<db_username>:<db_password>@postgres-postgresql.default.svc.cluster.local:5432/<db_name>
4. Запустите команды  
```shell-session
$ kubectl create configmap django-env --from-env-file=django.env
$ kubectl apply -f django_deploy.yaml
$ helm install postgres\
 --set auth.username=<db_username>\
 --set auth.password=<db_password>\
 --set auth.database=<db_name>\
 bitnami/postgresql
```
Имя базы, имя и пароль пользователя должны соответствовать DATABASE_URL из шага 3.  
5. Запустите миграцию БД:
```shell-session
$ kubectl apply -f migrate-job.yaml
```
6. В файл /etc/hosts добавьте строку `127.0.0.1 star-burger.test`
7. [Проверьте работу сайта](http://star-burger.test) 