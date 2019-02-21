# Сборка и Запуск ASP.NET WebForms Приложения в Docker

---

<section data-background-image="https://github.com/sixeyed/docker-windows-workshop/blob/dceu18/slides/img/frontend/Slide1.PNG?raw=true">

---

Наше демо приложение - простое ASP.NET WebForms приложение, которое использует SQL Server для хранения данных. Приложение целиком написано на .NET Framework `4.7.2`.

Приложение является монолитом. К концу воркшопа мы разобьем его на отдельные компоненты, но сначала нам нужно его запустить.

---

## Соберем Docker image для приложения

Давайте посмотрим на [Dockerfile](https://github.com/akamenev/docker-windows-workshop/blob/master/docker/frontend-web/v1/Dockerfile) для приложения. Она использует Docker, чтобы собрать приложение и упаковать его в образ (image).

_Соберите Docker image:_

```
cd $env:workshop

docker image build -t dwwx/signup-web `
  -f .\docker\frontend-web\v1\Dockerfile .
```

---

## Соберем образ получше

Первая версия Dockerfile простая, но не очень эффективная. Вторая версия [v2 Dockerfile](https://github.com/sixeyed/docker-windows-workshop/blob/master/docker/frontend-web/v2/Dockerfile) разделяет NuGet restore и MSBuild части - это позволяет осуществлять сборки быстрее.

_Соберите image:_

```
cd $env:workshop

docker image build -t dwwx/signup-web:v2 `
  -f .\docker\frontend-web\v2\Dockerfile .
```

---

## Запустите приложение

На этом все! 

Вам не нужны ни Visual Studio ни .NET 4.7.2, чтобы собрать приложение, все что вам нужно - исходный код и Docker. 

_Попробуйте запустить приложение в контейнере:_

```
docker container run `
  -d -p 8020:80 --name app `
  dwwx/signup-web:v2
```

---

## Пейредите в приложение

Вы можете перейти по порту `8020` на самой виртуальной машине с Docker. Или же вы можете перейти по адресу самого контейнера напрямую:

_Получите IP адрес контейнера и запустите браузер:_

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app

firefox "http://$ip/app"
```

---

## Удалите контейнер, прежде чем мы попробуем снова

Упс. 

Приложению нужен SQL Server, а его на машине нет. Мы его запустим далее, но сперва нужно удалить запущенный контейнер.

_Удалите `app` контейнер:_

```
docker container rm -f app
```

---

## Запустите приложение, на этот раз с зависимостями

Теперь мы запустим БД сервер в контейнере, используя Docker Compose для управления всем приложением целиком. Посмотрите на [v1 manifest](https://github.com/akamenev/docker-windows-workshop/blob/master/app/v1.yml), он описывает SQL Server и веб-приложение. 

_Запустите приложение, используя compose:_

```
docker-compose -f .\app\v1.yml up -d
```

---

## Проверьте, что все работает

Теперь у нас запущено 2 контейнера. В одном находится приложение, которое мы собрали из исходников, во втором находится SQL Server, созданный из публичного Docker образа от Microsoft

_Выведите список всех запущенных контейнеров:_

```
docker container ls
```

---

## Перейдите в веб-приложение

Как и ранее, перейдите по порту `8020` на виртуальной машине или перейдите напрямую по адресу контейнера:

```
$ip = docker container inspect `
  --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_signup-web_1

firefox "http://$ip/app"
```

---

## Выглядит лучше 

Но давайте проверим, что оно действительно работает. Нажмите кнопку _Sign Up_ , заполните форму и нажмите _Go!_ чтобы сохранить данные о себе.

_Проверьте, что данные были сохранены в SQL контейнере:_

```
docker container exec app_signup-db_1 `
  powershell `
  "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database SignUp"
```

---

## All good

Выглядит отлично. Это могло быть старое WebForms приложение, написанное 10 лет назад. Теперь вы можете его упаковать в контейнер и переместить в облако - **без изменений кода**!

Это также отличная отправная точка для модернизации приложения.
