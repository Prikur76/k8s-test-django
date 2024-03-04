# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

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

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Работа с локальным K8s кластером

### Для начала:
- скачать [VirtualBox](https://www.virtualbox.org/wiki/Downloads), [kubectl.exe](https://dl.k8s.io/release/v1.29.2/bin/windows/amd64/kubectl.exe), [minikube.exe](https://github.com/kubernetes/minikube/releases)
- создать директорию Kubernetes, добавить ее в Path переменных окружения Windows (если работаете в среде Windows)
- перенести файлы kubectl.exe, minikube.exe в директорию Kubernetes
- запустить кластер Minikube

### Команды для работы с кластером Minikube
- `minikube version` -- выводит версию Minikube
- `kubectl version` -- выводит версию kubectl

- запускает кластер Minikube c драйвером Virtualbox:
```bash 
minikube start --driver=virtualbox --no-vtx-check=true
``` 
- `minikube stop` -- останавливает кластер Minikube
- `minikube delete` -- удаляет кластер Minikube
- `minikube ip` -- выводит IP адрес кластера Minikube
- `minikube dashboard --url` -- запускает дашборд кластера Minikube и выводит URL 
- `minikube image ls --format=table` -- выводит список образов в кластере Minikube в табличном виде

### Команды для сборки образа с переменными окружения
- собирает образ из Dockerfile, с переменными окружения:
```bash
minikube image build -t django_app . -f ./Dockerfile \
   --build-env="SECRET_KEY=REPLACE_ME" \
   --build-env="ALLOWED_HOSTS=['127.0.0.1','localhost','*']" \
   --build-env="DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@192.168.0.246:5432/test_k8s"
``` 
- `minikube image rm <Image ID>` -- удаляет образ с указанным ID
- `minikube image prune -a` -- удаляет все неиспользуемые образы в кластере Minikube

### Запуск образа приложение в режиме [терминала](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#run):
```bash
kubectl run django \
--image=django_app:latest \
--image-pull-policy=IfNotPresent \
--env="DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@192.168.0.246:5431/test_k8s"
```
`kubectl exec django -ti -- bash` -- вход в терминал POD django

### Подключение к БД PostgreSQL
- запуск БД Postgres в отдельном контейнере, перед которым проверьте включение Docker Desktop:
```bash
docker-compose -f docker-compose.override.yml up -d
```
- ввести вручную настройки для подключения к базе данных:
```bash
export DATABASE_URL="postgres://test_k8s:OwOtBep9Frut@<HOST IP>:5431/test_k8s"
```
- провести миграцию, создать суперпользователя и/или проверить соединение с БД:
```
./manage.py migrate

./manage.py createsuperuser

./manage.py shell

>>> from django.contrib.auth.models import User
>>> User.objects.all()
```
Пример вывода:
```<QuerySet [<User: admin>]>```

### Создание объектов из Манифестов
- команда для создания объекта из Манифеста: ```kubectl apply -f <path-to-manifest>```

Например, создаем POD с помощью Манифеста:
```bash
kubectl apply -f ./kubernetes/pod.yaml
```
- Для сохранения объекта (POD, Service, Deployment) в файл Манифеста используйте команду: 
```bash
kubectl get pod django -o yaml > ./kubernetes/pod.yaml
```

### Развертывание 
- поместите в файл deployment.yaml инструкции для создания Deployment, Service, Pod, разделив их между собой пустой строкой и символами "--".
- запускаем создание объектов из файла deployment.yaml
```bash
kubectl apply -f ./kubernetes/deployment.yaml
```
- при необходимости - меняем настройки и повторно запускаем создание объектов из файла deployment.yaml
