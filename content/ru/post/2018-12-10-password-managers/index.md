---
title: Выбираем open source менеджер паролей
archives: "2018"
layout: post
tags: [opensource,  bitwarden, passwordmanager]
---
После того, как на работе был заблокирован Dropbox и для [синхронизации базы паролей пришлось городить костыли](/2018/08/06/check-password-database-async.html), я крепко задумался о выборе более подходящего варианта.

К менеджеру паролей у меня несколько требований:

* Open Source -  можно быть уверенным, что данные никуда не отправляются (в теории, можно почитать исходный код и убедиться в качестве кода).
* Self-hosted - можно самому держать сервер с паролями, а с разных клиентов(телефон, ноут) просто брать свежую версию.
* Мобильный клиент, плагин для браузера, приложение на компьютер - так как пароли используются .

Вариантов оказалось не много, но в итоге я нашёл что хотел.
<!--more-->

## Open Source

Сразу скажу, что вариант 1Password, LastPass или другие облачные решения - не рассматривались. Потому что хостятся непонятно где и в них периодически находят уязвимости ( LastPass [ раз ](https://habr.com/post/357304/) и [ два ](https://habr.com/company/defconru/blog/260383/)). Поэтому изначально пользовался [ KeePass ](https://keepass.info/) на Windows, потом [ KeePassX ](https://www.keepassx.org/) на линукс и мак.

Как оказалось, KeePass прошёл множество проверок на безопасность и [ рекуммендуется EU Free and Open Source Software Auditing (EU-FOSSA) ](https://keepass.info/ratings.html). Единственный его минус - он работает только на Windows. Для того, чтобы запустить на Linux или MacOS можно воспользоваться второй версией, но для этого нужно установить Mono - свободную реализацию .NET Framework для UNIX-подобных систем. При работе с Mono иногда вылезают разные косяки, поэтому было решено попробовать [ неофициальные порты ](https://keepass.info/download.html)(находятся чуть ниже на странице).

Я пользовался KeePassX, который меня всем устраивал. Он спокойно читал файлы KeePass 1й и 2й версии. Файлы данных синхронизировались через дропбокс и жил я в общем-то спокойно, пока [ дропбокс не был заблокирован ](/2018/08/06/check-password-database-async.html).

Тогда посмотрел какие есть варианты Open Source менеджеров паролей с возможностью хостить хранилище где-нибудь на сервере.

## Self-hosted

Оказалось, что под это требование попадает достаточно немного вариантов. Вот удобный [ список из wikipedia ](https://en.wikipedia.org/wiki/List_of_password_managers#Basic_Information).

Варианта 2:
1. Mitro - был закрыт 6 октября 2015 через год после того, как был куплен Twitter-ом, соответственно не подходит.
2. BitWarden - Кроссплатформенный облачный сервис, с десктопными и мобильными приложениями, а также с дополнениями к большинству браузеров. Так что он сразу подошёл и под 3-е требование:


## Мобильный клиент, плагин для браузера, приложение на компьютер

Bitwarden - это web-сервер с API, с которым можно работать из браузера, мобильного и десктопного приложения. О нём я узнал из одного выпуска [ Radio-T ](https://radio-t.com/p/2018/07/14/podcast-606/), но сразу перебираться на этот менеджер паролей не торопился, решил присмотреться, почитать о нём.

Когда всё-таки решился установить к себе на сервер, полез в документацию за [ инструкциями по установке ](https://help.bitwarden.com/article/install-on-premise/) и несколько раз был сильно удивлён:

1. Серверная часть написана на .NET. Похоже, это основной инструмент для разработки менеджеров паролей. В принципе разницы особой нет, ведь можно запустить в контейнере.
2. [ Установка и настройка ](https://help.bitwarden.com/article/install-on-premise/) производятся одним большим файлом, который нужно скачать и запустить. В это время скачиваются 6 образов docker, настраивается docker-compose и запускаются сервисы автоматически. Подход современный и даже хипстерский, но я предпочитаю автоматизировать в своих скриптах, чтобы при переезде на другой сервер можно было легко восстановить нужное состояние.
3. База данных для хранения паролей - MSSQL. Блин, вы серьёзно? Но дело не в базе, а в её требованиях. Оказалось, что минимально для её одной нужно минимум 4 ГБ ОЗУ, которых на сервере не было.
4. Настройка веб-сервера(nginx) и сертификата. У меня и так есть nginx и с сертификатами тоже как-нибудь разберусь.

Учитывая всё вышеперечисленное я сильно засомневался хочу ли я возиться со всем этим, и после продолжительного поиска оказалось, что добрые люди имплементировали такой же серверный функционал(API) на [Rust](https://github.com/dani-garcia/bitwarden_rs), [Ruby](https://github.com/jcs/rubywarden), [ Golang ](https://github.com/VictorNine/bitwarden-go) а в качестве базы данных используется чаще всего SQLlite3, что намного менее требовательно по ресурсам и проще в поддержке.

Как я это всё собирал и настраивал хочу написать позже.

## Выводы

**Плюсы**:
- Open Source (как официальный, так и неофициальные имплементации)
- Self-Hosted - простой веб сервер.
- Есть плагины для браузеров, мобильное приложение и для десктопа.
- Надёжность. Оказалось, что bitwarden [ проходил аудит по безопасности ](https://blog.bitwarden.com/bitwarden-completes-third-party-security-audit-c1cc81b6d33)(по крайней мере официальный сервер, отчёт можно найти в этом посте)

**Минусы**:
- Муторность установки и запуска официального клиента. Тяжело поддаётся автоматизации.
- Непривычный интерфейс и сочетания клавиш(А в десктопном приложении для MacOS они просто отсутствуют)

Bitwarden - совпал по всем моим требованиям и в принципе я доволен его работой. Тем кто интересуется этим вопросом рекомендую обратить внимание на этот продукт.
