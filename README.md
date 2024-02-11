# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет
сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и
Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и
Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его
установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции
будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по
адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не
требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции
схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие
миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой
библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно
лишь, чтобы оно никому не было
известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE`
или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт
ответит ошибкой 400. Можно перечислить несколько адресов через запятую,
например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не
поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Деплой [Kubernetes](https://kubernetes.io/) кластера с помощью [Minikube](https://minikube.sigs.k8s.io/docs/)

1) Установить [`Kubectl`](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/), [`VirtualBox`](https://www.virtualbox.org/), [`Minikube`](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/), [`Helm`](https://helm.sh/) запустить minikube:

```sh
$ minikube start
```

2) Включите [Ingress](https://habr.com/ru/companies/slurm/articles/358824/)

Выберете и установите нужный Ingress, согласно [документации](https://docs.google.com/spreadsheets/d/191WWNpjJ2za6-nbG4ZoUMXMpUK8KlCIosvQB0f-oq3k/edit#gid=907731238)

```sh
$ minikube addons enable ingress
$ kubectl apply -f k8s-yaml/ingress_example.yaml
```

3) Пропишите в `etc/hosts` ip minikube:

Узнайте `ip address minikube`:

```sh
$ minikube ip
```
В файл `hosts` добавить(после этого можно будет заходить на сайт `star-burger.test` вместо цифр):
```
...
192.168.1.1 star-burger.test www.star-burger.test
...
```

4) Загрузите образ `backend_main_django` в [DockerHub](https://hub.docker.com/)

5) Разверните БД в контейнере. Вот [несколько способов](https://yeah366.com/2023/01/How-to-deploy-PostgreSQL-in-Kubernetes/#45_kubectl__PostgreSQL_557) для деплоя. Пример:

```sh
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
$ helm install psql-test1 bitnami/postgresql
$ export POSTGRES_PASSWORD=$(kubectl get secret --namespace default psql-test1-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
$ kubectl run psql-test1-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.1.0-debian-11-r19 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host psql-test1-postgresql -U postgres -d postgres -p 5432
```

Создайте БД [Postgres](https://www.postgresql.org/):
```
CREATE DATABASE yourdbname;
CREATE USER youruser WITH ENCRYPTED PASSWORD 'yourpass';
GRANT ALL PRIVILEGES ON DATABASE yourdbname TO youruser;
ALTER USER youruser SUPERUSER;
\conninfo # Скопировать host db
```
Данные БД скопировать и заменить в файле `k8s-yaml/secret_example.yaml`. Изменить SecretKey django:
```
...
DATABASE_URL: "postgres://<youruser>:<yourpass>@<yourhost>:5432/<yourdbname>"
SECRET_KEY: "REPLACE_me_pls"
...
```

6) Сохраните [Secret](https://kubernetes.io/docs/concepts/configuration/secret/), примените миграции и задеплойте сервис django:

```sh
$ kubectl apply -f k8s-yaml/configmap_example.yaml
$ kubectl apply -f k8s-yaml/secret_example.yaml
$ kubectl apply -f k8s-yaml/migrate_example.yaml
$ kubectl apply -f k8s-yaml/deploy_example.yaml
$ kubectl apply -f k8s-yaml/service_example.yaml
```

7) Автоматическое удаление сессий Django-приложения(1 числа каждого месяца в 00:00) c помощью [CronJobs](https://tproger.ru/translations/guide-to-cron-jobs):

```sh
$ kubectl apply -f k8s-yaml/django_clearsession_pod_example.yaml
```


