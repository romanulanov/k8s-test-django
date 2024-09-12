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

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

### Как добавить секретные данные 

Для того, чтобы задать секретные данные используйте secret.yaml [подробнее здесь](https://kubernetes.io/docs/concepts/configuration/secret/). 
Шаги создания секрета:
1. Закодируйте данные  в Base64. Например для secret_key="1234qwerty": 

```bash
echo -n "1234qwerty" | base64
```

2. Полученные значения запишите в yaml-файл:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=    # Это значение "admin" в Base64
  password: czNjcjN0    # Это значение "s3cr3t" в Base64
```

3. Примените YAML-манифеcт:

```bash
kubectl apply -f secret.yaml
```

4. После создания секрета используйте его в поде, например:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: test-container
    image: nginx
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### Развёртывание в minikube

Установите и настройте локальный k8s кластер. [Подробнее тут.](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
Запустите Minikube с помощью команды:

```bash
minikube start
```
После того, как зададите секретные ключи, примените манифест, настройте Deployment и накатите миграции:

```bash
kubectl apply -f secret.yaml
kubectl apply -f django-deployment.yml
kubectl apply -f django-service.yml
kubectl apply -f django-ingress.yml
kubectl apply -f django-migrate.yaml
```

Для регулярного удаления устаревших сессий примените:

```bash
kubectl apply -f django-clearsessions.yaml
```

Результат развёртывания можно посмотреть с помощью команды:

```bash
minikube service django-service
```


### Как подготовить dev окружение

Для работы в кластере с PostgreSQL для начала нужно получить ssl-сертификат. [Инструкции по подключению к базе данных Managed PostgreSQL Яндекс Облака (Раздел “Получение SSL-сертификата”).](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect) После этого создадим секрет сертификата в кластере с помощью команды:

```bash
kubectl create secret generic postgres-cert  --from-file=ssl-cert=RootCA.pem  --namespace=<namespace>
```