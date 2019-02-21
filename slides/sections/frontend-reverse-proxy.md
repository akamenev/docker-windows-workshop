# Выделение Homepage из приложения

---

<section data-background-image="https://github.com/akamenev/docker-windows-workshop/blob/master/slides/img/frontend/Slide2.PNG?raw=true">

---

Монолиты могут прекрасно работать в контейнерах, но от этого они не становятся "modern apps" - они просто старые приложения, запущенные в контейнерах.

Вы можете переделать монолитное приложение в микросервисное, но это достаточно длительный процесс. 

Вместо этого, мы будем идти поступательно, выделяя функционал из монолита и запуская его в отдельных контейнерах. Начнем мы с домашней страницы нашего приложения.

---

## Новая домашняя страница

Посмотрите [новую домашнюю страницу](https://github.com/akamenev/docker-windows-workshop/blob/master/docker/frontend-reverse-proxy/homepage/index.html). Это статичный HTML сайт, который использует Vue.js - он будет работать в отдельном контейнере, что позволяет использовать другой стэк технологий, отличный от основного приложения.

[Dockerfile](https://github.com/akamenev/docker-windows-workshop/blob/master/docker/frontend-reverse-proxy/homepage/Dockerfile) очень простой - он просто копирует HTML в образ с IIS.

_Соберите образ с новой домашней страницей:_

```
docker image build `
  -t dwwx/homepage `
  -f .\docker\frontend-reverse-proxy\homepage\Dockerfile .
```

---

## Запустите новую домашнюю страницу

Теперь вы можете запускать домашнюю страницу отдельно, что позволяет очень быстро вносить изменения. 

_Запустите homepage:_

```
docker container run -d -p 8040:80 --name home dwwx/homepage
```

---

## Попробуйте на нее перейти

Домашняя страница доступна на порту `8040` вашей виртуальной машины, вы можете перейти туда или по прямому адресу контейнера:

_Получите IP адрес контейнера и запустите браузер:_

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' home

firefox "http://$ip"
```

---

## Почти готово

Новая домашняя страница выглядит отлично, быстро запускается и упакована в небольшой образ с Nano Server.

Однако она не работает сама по себе - нажмите _Sign Up_ и вы получите ошибку.

Чтобы использовать новую домашнюю страницу **без изменений изначального приложения** мы можем запустить reverse-proxy в отдельном контейнере.

---

## The reverse proxy

Мы будем использовать [Nginx](http://nginx.org/en/). Все запросы будут приходить на Nginx, а он будет перенаправлять запросы в зависимости от ситуации.

Nginx позволяет сделать больше - в [конфигурационном файле nginx.conf]https://github.com/akamenev/docker-windows-workshop/blob/master/docker/frontend-reverse-proxy/reverse-proxy/conf/nginx.conf) мы задаем параметры для кэширования, а также можем настроить SSL-temination

_Соберите образ reverse proxy:_

```
docker image build `
  -t dwwx/reverse-proxy `
  -f .\docker\frontend-reverse-proxy\reverse-proxy\Dockerfile .
```

---

## Обновитесь для использования новой домашней страницы

Посмотрите на [v2 manifest](https://github.com/akamenev/docker-windows-workshop/blob/master/app/v2.yml) - он добавляет новые сервисы - homepage и reverse-proxy

Только в прокси указаны порты. Это публичная точка входа для приложения, остальные контейнеры могут видеть друг друга, но извне их не видно.

_Обновитесь до v2:_

```
docker-compose -f .\app\v2.yml up -d
```

> Compose сравнивает текущее состояние с желаемымым (определенном в манифесте) и запускает новые контейнере. 

---

## Попробуйте новое приложение

Reverse proxy опубликована на порту `8020`, можете перейти по нему или по адресу контейнера с Nginx:

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_proxy_1

firefox "http://$ip"
```

> Теперь вы можете перейти на страницу с _Sign Up_.

---

## Ну и давайте проверим

Давайте проверим, что ничего не сломалось

Нажмите на _Sign Up!_, заполните форму и нажмите _Go!_ для сохранения данных.

_Проверьте, что новые данные появились в контейнере с SQL:_

```
docker container exec app_signup-db_1 powershell `
  "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database SignUp"
```

---

## Все отлично

Теперь у нас есть reverse-proxy, которая позволяет нам отделить UI часть от монолита. 

Мы запустили новую домашнюю страницу на Vue, но мы могли бы использовать CMS для домашней страницы или могли бы поменять форму Sign Up на что-то другое.

Все эти маленькие части могут быть развернуты независимо, а также масштабированы. Все это облегчает задачу выпуска новых релизов фронтэнд части приложения без необходимости редеплоя и осуществления регресионного тестирования всего монолита.
