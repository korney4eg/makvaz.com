---
layout: post
title: Просто анализируем посещаемость с помощью goaccess
archives: "2019"
featureImg: "assets/img/configuring-goaccess/user_activity.png"
thumbnail: "assets/img/configuring-goaccess/user_activity.png"
tags: [goaccess, сайт, howto]
aliases:
  - /2019/02/06/configuring-goaccess.html
---
Для того, чтобы анализировать пользовательскую активность на своём блоге или сайте необязательно пользоваться гугл аналитикой, яндекс метрикой или другими внешними сервисами. Всё что нужно находиться у вас на сервере, а конкретно - **access-логи**. Из них можно узнать сколько пользователей пришло на ваш сайт, откуда они пришли, во сколько и много-много другого. Анализировать логи можно вручную, можно с помощью своей программы или воспользоваться программой, которая под это дело заточена.

В сети я нашёл три более-менее вменяемые программы:

- [AWStats](https://awstats.sourceforge.io/) - была довольно популярной раньше, но к сожалению дизайн так и остался из 1998 года([ демо ](https://www.nltechno.com/awstats/awstats.pl?config=destailleur.fr))
- [Open Web Analytics](http://www.openwebanalytics.com/) - даёт слишком много данных, и не заточена под мобильные браузеры([ демо ](http://demo.openwebanalytics.com/owa/index.php?owa_siteId=c9b7d12e322c7c360fb8f7c72ffe4c41&owa_period=last_seven_days&owa_do=base.reportDashboard))
- [goaccess](https://goaccess.io/) - строит 14 видов графиков(на момент написания статьи), хорошо просматривается на мобильном и активно поддерживается.([ демо ](https://rt.goaccess.io/?20181123091935))

Вот о том, как поставить и настроить под себя goaccess и пойдёт речь в этой статье.
<!--more-->

goAccess может строить следующие графики:

1. Пользователи и клики за день(**UNIQUE VISITORS PER DAY**)
2. Запросы(**REQUESTED FILES (URLS**))
3. запрашиваемые статичные файлы(**STATIC REQUESTS**)
4. Не найденные страницы(**NOT FOUND URLS (404S)**)
5. Ip адреса(**VISITOR HOSTNAMES AND IPS**)
6. Операционные системы(**OPERATING SYSTEMS**)
7. Браузеры(**BROWSERS**)
8. Время посещения(**TIME DISTRIBUTION**)
9. Реферальные ссылки, с которых пришли посетители(**REFERRERS URLS**)
10. Сайты, с которых пришли посетители(**REFERRING SITES**)
11. Поисковые запросы в гугле(**KEYPHRASES FROM GOOGLE'S SEARCH ENGINE**)
12. Статус коды(**HTTP STATUS CODES**)
13. Геолокация - География посетителей(**GEO LOCATION**)

Уже достаточно много, хотя после настройки останутся не все, только нужные.

## Установка

Проще всего goaccess установить вашим любимыми [ менеджером пакетов ](https://github.com/allinurl/goaccess#distributions), однако в этой поставке не все опции подключены. Мне, например, очень интересна география пользователей, откуда они пришли(геолокация). Поэтому goaccess нужно собирать с дополнительными флагами. Ещё, я запускаю goaccess на разных системах: дома на макбуке и на сервере на ubuntu, поэтому больше всего мне понравился вариант запуска в docker-контейнере. Весить наш образ будет примерно 140 Mb, зато запускать можно будет на любой системе.

В стандартном [образе](https://hub.docker.com/r/allinurl/goaccess) не хватает работы с геолокацией, поэтому он тоже был отброшен.

Прежде всего клонируем самую свежую версию goaccess и создадим директории, куда мы будем закидывать конфигурационный файл, файл логов, и где будет сохраняться отчёт:

```bash
git clone https://github.com/allinurl/goaccess
cd goaccess
mkdir -p srv/{data,logs,html}
```

Приводим *Dockerfile* к следующему виду:

```
FROM alpine:edge

COPY . /goaccess
WORKDIR /goaccess

ARG build_deps="build-base ncurses-dev autoconf automake git gettext-dev libmaxminddb-dev"
ARG runtime_deps="tini ncurses libintl gettext openssl-dev libmaxminddb"

RUN apk update && \
    apk add -u $runtime_deps $build_deps && \
    autoreconf -fiv && \
    ./configure --enable-utf8 --with-openssl  --enable-geoip=mmdb && \
    make && \
    make install && \
    apk del $build_deps && \
    mkdir -p /usr/share/GeoIP/ && \
    cd /usr/share/GeoIP && \
    wget  https://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz && \
    tar -zxvf GeoLite2-City.tar.gz && \
    mv GeoLite2-City_*/GeoLite2-City.mmdb /usr/share/GeoIP/ && \
    rm -rf GeoLite2-City_*  GeoLite2-City.tar.gz && \
    rm -rf /var/cache/apk/* /tmp/goaccess/* /goaccess

VOLUME /srv/data
VOLUME /srv/logs
VOLUME /srv/report
EXPOSE 7890

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["goaccess", "--no-global-config", "--config-file=/srv/data/goaccess.conf"]
```

Что мы тут делаем: 
1. устанавливаем все зависимости goaccess
2. конфигурируем с поддержкой геолокации
3. собираем и устанавливаем goaccess
4. скачиваем и устанавливаем [базу геоданных](https://dev.maxmind.com/geoip/geoip2/geolite2/)
5. чистим всё лишнее
6. Создаём точку монтирования конфигурационного файла
7. Создаём точку монтирования папки с логами
8. Создаём точку монтирования папки, куда будет сохраняться отчёт

Собираем в докер образ:

```bash
docker build -t myuser/goaccess:latest .
```

Всё, теперь его можно запускать как сервис или как программу с нужными параметрами.

Для примера, скопируем лог файл как *srv/logs/access.log* и запустим goaccess.

```bash
docker run   -v "/Users/myuser/projects/goaccess/srv/logs/:/srv/logs" -v "/Users/myuser/projects/goaccess/srv/html/:/srv/report" myuser/goaccess:latest goaccess -f /srv/logs/access.log  --log-format=COMBINED -o /srv/report/index.html
```

*/Users/myuser/projects/goaccess/srv/\** - полный путь до директорий, которые мы создавали в начале установки.(для докера нужно указывать только полный путь) Понятно, что у вас эти пути будут свои.

После запуска отчёт можно найти по адресу *srv/html/index.html*

## Настройка

Настройки и параметры можно закинуть в конфигурационный файл *srv/data/goaccess.conf*:
```conf
log-format COMBINED
log-file /srv/logs/access.log
output /srv/report/index.html
anonymize-ip true
ignore-panel REQUESTS_STATIC
ignore-panel NOT_FOUND
ignore-panel VIRTUAL_HOSTS
ignore-panel KEYPHRASES
ignore-panel STATUS_CODES
ignore-panel REMOTE_USER
real-os true
geoip-database /usr/share/GeoIP/GeoLite2-City.mmdb
```
где,

**log-format** - формат логов. Для nginx и apache httpd = COMBINED

**log-file** - путь к файлу с логами

**output** - куда сохраняем отчёт

**anonymize-ip** - маскируем ip адреса

**ignore-panel** - скрываем панели

**real-os** - показывать более полное название OS(Windows XP, вместо Windows NT 5.1)

**geoip-database** - путь к базе адресов.

Полный конфигурационный файл можно найти [ здесь ](https://github.com/allinurl/goaccess/blob/master/config/goaccess.conf)

Запуск выглядит попроще:

```bash
docker run   -v "/Users/myuser/projects/goaccess/srv/logs/:/srv/logs" -v "/Users/myuser/projects/goaccess/srv/data/:/srv/data" -v "/Users/myuser/projects/goaccess/srv/html/:/srv/report" myuser/goaccess:latest
```

## Заключение

В итоге у меня получился [вот такой отчёт](/assets/html/usage_report.html).

Теперь наш образ можно залить на Docker Hub и пользоваться им на удалённом сервере.

Я регулярно(каждую неделю) строю посещаемость на сайте и отправляю в telegram.

## P.S.:

Goaccess позволяет работать как сервис и строить графики в реальном времени(пример ссылка). Пока мне это не нужно было, но если вам интересно, посмотрите [ сюда ](https://goaccess.io/man#examples)(поищите REAL TIME HTML OUTPUT)
