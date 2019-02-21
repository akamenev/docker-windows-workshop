# Выделение Data Reference API

---

<section data-background-image="https://github.com/akamenev/docker-windows-workshop/blob/master/slides/img/backend/Slide1.PNG?raw=true">

---

Docker позволяет запускать отдельные фичи в контейнерах, а также обеспечивает коммуникацию между контейнерами.

Сейчас наше приложение загружает референсные данные напрямую из базы данных - список стран и список ролей из выпадающих списков.

Мы это поменяем и будем отдавать эти списки через Data Reference API.

---

## The reference data API

Новый компонент это простой REST API. Вы можете посмотреть [исходный код для Reference Data API](https://github.com/akamenev/docker-windows-workshop/tree/master/src/SignUp.Api.ReferenceData) - в нем содержится один контроллер для стран и другой для ролей.

API использует новый технологический стэк:

- [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/?view=aspnetcore-2.1) как более быстрая и кросс-платформенная альтернатива ASP.NET
- [Dapper](https://github.com/StackExchange/Dapper) как быстрый и легковесный ORM

Мы можем использовать эти новые технологии не затрагивая при этом монолит, потому что эти компоненты работают в отдельных контейнерах

---

## Соберите API

Посмотрите [Dockerfile](https://github.com/akamenev/docker-windows-workshop/blob/master/docker/backend-rest-api/reference-data-api/Dockerfile) для API. 

Он использует тот же принцип для сборки и упаковки приложения, только используются образы .NET Core.

_Соберите API image:_

```
docker image build `
  -t dwwx/reference-data-api `
  -f .\docker\backend-rest-api\reference-data-api\Dockerfile .
```

---

## Запустите новый API

Вы можете запустить API сам по себе, но ему нужен доступ в SQL Server. 

В образе уже есть connection string по умолчанию, которую мы можем перезаписать с помощью переменных окружения.

_Запустите API с соединением к контейнеру с SQL:_

```
docker container run -d -p 8060:80 --name api `
  -e ConnectionStrings:SignUpDb="Server=signup-db;Database=SignUp;User Id=sa;Password=DockerCon!!!" `
  dwwx/reference-data-api
```

---

## Перейдите на API

API доступен по порту `8060` на виртуальной машине, можете перейти туда или напрямую в контейнер:

_Получите IP контейнера и запустите браузер:_

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' api

firefox "http://$ip/api/countries"
```

> Поменяйте `/countries` на `/roles` чтобы увидеть другой датасет

---

## Обновите приложения для использования API

Теперь мы можем запустить приложение с использованием нового API. Посмотрите на [v3 manifest](https://github.com/akamenev/docker-windows-workshop/blob/master/app/v3.yml) - он добавляет REST API.

Манифест также конфигурирует web app для использования API.

_Обновитесь до v3:_

```
docker-compose -f .\app\v3.yml up -d
```

---

## Проверьте что запущено

В данный момент уже запущено много контейнеров - первоначальное приложение и база данных, новая домашняя страница, reverse-proxy и новый API

_Выведите список запущенных контейнеров:_

```
docker container ls
```

---

## Попробуйте новое распределенное приложение

Входная точка все еще прокси, работающая на порту  `8020`, можете перейти туда или по адресу контейнера:

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_proxy_1

firefox "http://$ip"
```

> Теперь когда вы нажимаете на _Sign Up_ page, выпадающие списки подгружаются из API.

---

## Давайте проверим это

Новый REST API логгирует обращения в консоль.

Логи покажут, что контроллеры для стран и ролей были вызваны веб-приложением

_Посмотрите логи:_

```
docker container logs app_reference-data-api_1
```

---

## Ну и давайте убедимся наверняка

Нажмите _Sign Up!_, заполните форму и нажмите _Go!_ чтобы сохранить данные.

_Проверьте,что данные появились в SQL контейнере:_

```
docker container exec app_signup-db_1 powershell `
  "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database SignUp"
```

---

## Все ок

Теперь у нас есть небольшой, быстрый REST API, предоставляющий нам референс данные. Он доступен только для веб приложения, но мы можем сделать его доступным публично.

Как? Добавлением правила машрутизации в наш reverse-proxy, который был развенут ранее. Мы можем перенаправлять запросы, которые приходят на `/api` в контейнер с API.

Можете попробовать сделать это сами.

> Подсказка: стоит начать с `location` блоков в `nginx.conf`
