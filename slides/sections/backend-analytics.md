# Добавление Self-service аналитики

---

<section data-background-image="https://github.com/akamenev/docker-windows-workshop/blob/master/slides/img/backend/Slide3.PNG?raw=true">

---

Приложение использует SQL Server для хранения, который не всегда удобно использовать для бизнес-аналитики. Следующим шагом мы добавим сервис аналитики.

Мы запустим [Elasticsearch](https://www.elastic.co/products/elasticsearch) для хранения и [Kibana](https://www.elastic.co/products/kibana) для отображения аналитики. 

---

## Pub-sub messaging

Для того, чтобы отправлять данные с каждой новой регистрацией в Elasticsearch, нам нужен отдельный обработчик сообщений. 

Новый обработчик сообщений это .NET Core консольное приложение. Исходный код находится в файле [QueueWorker.cs](https://github.com/akamenev/docker-windows-workshop/blob/master/src/SignUp.MessageHandlers.IndexProspect/Workers/QueueWorker.cs) - он подписывается на события, а затем обогащает данные и отправляет их в Elasticsearch.

---

## Соберите обработчик сообщений для аналитики


[Dockerfile](https://github.com/akamenev/docker-windows-workshop/blob/master/docker/backend-analytics/index-handler/Dockerfile) следует уже знакомому нам принципу, сначала собирает приложение, а затем запаковывает его.


_ Соберите образ: _


```
cd $env:workshop; `

docker image build --tag dwwx/index-handler `
  --file .\docker\backend-analytics\index-handler\Dockerfile .
```

---

## Запуск Elasticsearch в Docker

Команда Elasticsearch поддерживает образ для Linux, но пока что не для Windows. 

Достаточно легко собрать свой образ для Elasticsearch в Windows контейнере, но мы воспользуемся тем, который уже собран: `sixeyed/elasticsearch`.

[Dockerfile](https://github.com/sixeyed/dockerfiles-windows/blob/master/elasticsearch/nanoserver/sac2016/Dockerfile) скачивает Elasticsearch и устанвливает его поверх официального OpenJDK образа.

---

## Запуск Kibana в Docker

Тоже самое делаем и с Kibana.

Мы будем использовать образ `sixeyed/kibana`. 

[Dockerfile](https://github.com/sixeyed/dockerfiles-windows/blob/master/kibana/windowsservercore/ltsc2016/Dockerfile) скачивает и устанавливает Kibana, и собирает [startup script](https://github.com/sixeyed/dockerfiles-windows/blob/master/kibana/windowsservercore/ltsc2016/init.ps1) с параметрами по-умолчанию.

---

## Запускаем приложение с аналитикой

В [v5 manifest](https://github.com/akamenev/docker-windows-workshop/blob/master/app/v5.yml), ни один из существующих контейнеров не был заменен - их конфигурация не поменялась. Добавляются только новые контейнеры:

```
cd "$env:workshop"; `

docker-compose -f .\app\v5.yml up -d
```

---

## Обновите страницу в браузере

Вернитесь на страницу sign-up в браузере. **Это тот же IP адрес** потому что контейнер с приложением не был заменен. 

Добавьте другого пользователя и вы увидите, что данные появились в SQL Server, но теперь оба обработчика сообщений записали в лог, что они обработали сообщение.

---

## Проверьте, что данные сохранены

И логи в обработчиках сообщений:

 ```
docker container exec app_signup-db_1 powershell `
  "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database SignUp"; `

docker container logs app_signup-save-handler_1; `

docker container logs app_signup-index-handler_1
```

> Можете добавить несколько новых пользователей с разными ролями и странами, чтобы появилось больше данных в Kibana.

---

## Посмотрите данные в Kibana

Kibana также является веб-приложением, работающим в контейнере, работающее на порту 5601.

_Получите IP адрес контейнера с:_

```
$ip = docker container inspect --format '{{ .NetworkSettings.Networks.nat.IPAddress }}' app_kibana_1; `

firefox "http://$($ip):5601"
```

> Индекс в Elasticsearch называется `prospects`, выбрав его, вы можете увидеть данные в Kibana. 

---

## Zero-downtime deployment

Новая event-driven архитектура позволяет вам добавлять различные фичи без обновления основного монолитного приложения.

Нет необходимости проводить регресионный анализ для этого релиза, так как функционал аналитики никак не задевает оригинальное приложение.
