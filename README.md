# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell
docker compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell
docker compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
docker compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образ, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить dev-версию в minikube

Запустите базу данных:
```shell
docker compose up -d db
```

В отдельном терминале запустите minikube и tunnel 
(необходим для подключения к сервису в случае с minikube), 
оставьте окно с tunnel открытым:
```shell
minikube start
minikube tunnel
```

Соберите образ приложения в minikube, назовите его `web_django`:
```shell
eval $(minikube docker-env)
docker image build -t web_django backend_main_django/
```

Запустите сервис (он будет направлять запросы к нужным подам):
```shell
kubectl apply -f k8s/web-django-service.yml
```

Создайте ConfigMap: скопируйте файл `web-django-config.yml.template`, 
удалив суффикс `.template`. 
Заполните значения:
- `SECRET_KEY`: любое (см. выше)
- `DATABASE_URL`: в секции `host` должен быть внешний IP-адрес вашего компьютера. 
Получить его можно командой `ip a`.
- `ALLOWED_HOSTS`: имя хоста вашего сайта (или несколько через запятую).

Загрузите конфиг:
```shell
kubectl apply -f k8s/web-django-config.yml
```

Добавьте DNS record. В случае с версией для разработки на minikube, получите IP-адрес куба с помощью команды `minikube ip` и добавьте строчку с этим IP в файл `/etc/hosts`. Например:
```
192.168.49.2 star-burger.test
```


Запустите deployment и ingress:
```shell
kubectl apply -f k8s/web-django-deployment.yml
kubectl apply -f k8s/star-burger-ingress.yml
```

Запустите CronJob для очистки устаревших сессий Django:
```shell
kubectl apply -f k8s/web-django-clearsessions.yml
```

Перезапустить deployment на новых настройках, например при изменениях в ConfigMap:
```shell
kubectl rollout restart deployment
```


Готово! Сайт доступен в браузере по адресам, указанным в `ALLOWED_HOSTS`.
