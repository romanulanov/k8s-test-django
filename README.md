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

В качестве UI для терминала можно использовать [k9s](https://k9scli.io/), в качестве UI для k8s [Lens Desktop](https://k8slens.dev/)
Для запуска сервиса в yandex cloud необходимо:
1. Подключиться к кластеру Yandex cloud
   1. [Установите и инициализируйте интерфейс командной строки Yandex Cloud](https://yandex.cloud/ru/docs/cli/quickstart#install)
   2. [Добавьте учетные данные](https://yandex.cloud/ru/docs/managed-kubernetes/operations/connect#kubectl-connect) кластера Kubernetes в конфигурационный файл kubectl:
     ```
     yc managed-kubernetes cluster get-credentials --id catvr0kq2b7bn0v8k2r9 --external
     ```
2. Используйте утилиту kubectl для работы с кластером Kubernetes:
    ```
      kubectl get cluster-info
      kubectl get pods --namespace=<your-namespace>
    ```

### Описание кластера
 - Выделен домен edu-reverent-mestorf.sirius-k8s.dvmn.org. Запросы обрабатывает Yandex Application Load Balancer.
 - В кластере K8s создан отдельный namespace edu-reverent-mestorf.
 - В Yandex Managed Service for PostgreSQL создана база данных edu-reverent-mestorf. Доступы лежат в секрете K8s
 - В Yandex Application Load Balancer создан роутер edu-reverent-mestorf. Он распределяет входящие сетевые запросы на разные NodePort кластера K8s.
 -  Настроен редирект HTTP → HTTPS.
    Схема роутинга: https://edu-reverent-mestorf.sirius-k8s.dvmn.org/ → NodePort 30291

### Получение SSL-сертификата для подключения к базе данных PostgreSQL
Для работы в кластере с PostgreSQL для начала нужно получить ssl-сертификат. [Инструкции по подключению к базе данных Managed PostgreSQL Яндекс Облака (Раздел “Получение SSL-сертификата”).](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect) После этого создадим секрет сертификата в кластере с помощью команды:

```bash
kubectl create secret generic postgres-cert  --from-file=ssl-cert=RootCA.pem  --namespace=<namespace>
```

### Образ в docker registry
Необходимо разместить образ приложения в [dockerhub](https://hub.docker.com):
* Соберите образ:
```shell
docker build -t image_name:tag_name -f path/to/dockerfile .
```
* Тегируйте образ:
```shell
docker tag image_name:tag_name repo_name/image_name:tag_name
```
* Загрузите образ в Docker Hub:
```shell
docker push repo_name/image_name:tag_name
```
По умолчанию, во всех скриптах используется `latest` версия. Тэг можно поменять на необходимый.

## Запуск проекта в Yandex Cloud
1. После подготовки окружения, необходимо создать Secret с переменными окружения Django (данные о БД кластера можно взять из секрета K8s):
    Формат YAML-файла:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secret
    type: Opaque
    data:
      ALLOWED_HOSTS: <your-base64-encoded-hosts>
      DATABASE_URL: <your-base64-encoded-db_url>
      SECRET_KEY: <your-base64-encoded-secret_key>
      DEBUG: <your-base64-encoded-debug>
      DB_USER: <your-base64-encoded-db_user>
      DB_PASSWORD: <your-base64-encoded-db_password>
    ```

Применим этот секрет в кластере:
```
    kubectl apply -f .\secrets.yaml --namespace=edu-reverent-mestorf
```
2. Затем создаем Deployment и Service из нашего образа с Docker Hub:
    ```
    kubectl create --namespace=edu-reverent-mestorf -f dev-k8/django_deployment.yaml
    kubectl create --namespace=edu-reverent-mestorf -f dev-k8/django_service.yaml
    ```
  Теперь Django-проект доступен по адресу хоста вашего ALB, в моем случае https://edu-reverent-mestorf.sirius-k8s.dvmn.org/
3. Применим миграции и настроим автоудаление сессий:
    ```
    kubectl apply -f  --namespace=edu-reverent-mestorf dev-k8/django_migrate.yaml
    kubectl apply -f  --namespace=edu-reverent-mestorf dev-k8/django_clearsessions.yaml
    ```
4. Для проверки работоспособности сайта и БД, зайдите в Pod с помощью `kubectl` и создайте суперпользователя:
    ```
   python manage.py createsuperuser
   ```
   Также можете отсматривать ошибки в логах пода:
   ```
   kubectl logs <django-pod> -n <namespace> 
   ```
   