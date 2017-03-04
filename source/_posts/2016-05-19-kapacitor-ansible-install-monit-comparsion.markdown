---
layout: post
title: "Kapacitor: часть 1. Введение, сравнение с Monit, установка с Ansible и без, настройка"
date: 2016-05-19 00:47:52 +0500
comments: true
categories:
- monitoring
- alerting
- influxdata
- kapacitor
- ansible
- github
- monit

---

Несколько недель назад я начал разбираться с Kapacitor, попутно записывая свои действия. Конца разбирательствам было не видно, записей становилось все больше и накопилось на серию.

Речь пойдет о Kapacitor, последнеем слое из стека [TICK](https://influxdata.com/get-started/what-is-the-tick-stack/) от InfluxData, набора программ для сбора, отображения и обработке метрик.

Tl;dr: думаю, что Kapacitor нужен только тем, кто уже использует InfluxDB для сбора метрик. С установкой могут быть проблемы, если руки кривые.

А также небольшое замечание о том, [как делать Pull request'ы из браузера за 2 минуты](/blog/2016/05/19/kapacitor-ansible-install-monit-comparsion/#github-pull-request)


<img style="background:#1F242D" src="/images/2016-05/kapacitor.svg" />

<!-- more -->

Я уже настроил три слоя из стека: на серверах стоят агенты Telegraf, передают метрики в InfluxDB, их можно смотреть в виде графиков через Grafana (InfluxData предлагает свой Chronograf, но он сильно отстает от Grafana по функционалу на январь 2016 и вряд ли это изменится).

У этой схемы есть недостаток: чтобы узнать, что что-то идет не так, нужно зайти в Grafana и глазами найти это что-то. Это меня устраивает, когда я уже знаю, что сервер плохо себя чувствует.

Kapacitor нужен для уведомлений, алертинга. В 2 словах: это демон, который умеет пропускать через себя данные, приходящие в InfluxDB, обрабатывать их и пересылать по разным каналам связи / на HTTP / в базу данных.

Для меня Kapacitor - прямой конкурент Monit, поэтому сравниваю с ним, больше ни с чем подобным дел не имел, но слышал, что для мониторинга серверов правильные пацаны используют Zabbix, Nagios/Icinga, Sensu, Riemann. Я решил пока не добавлять софта на сервера, да и уведомлять на основе уже собранных данных мне кажется правильным, этим объясняется мой выбор в пользу Kapacitor.

### Плюсы Kapacitor
1. Убирание лишнего. Kapacitor не надо ставить агентом, роль агента выполняет Telegraf. Monit, которым я пользуюсь сейчас для алертинга, дублирует функционал, собирая метрики самостоятельно.
2. Надежный алертинг. У monit тут есть проблема: когда умирает сервер, monit, установленный там, тоже умирает и не успевает отправить алерт на email. Надежный, кроме случаев, когда падает Kapacitor или InfluxDB, что случается.
3. Продвинутый алертинг. Monit умеет мало (ладно, много, но я умею на нем мало). Kapacitor имеет в распоряжении данные всех моих серверов, что позволяет ему смотреть на них как на систему. У меня в этом месте фантазия начинает играть, не буду расписывать, что по моему мнению можно отслеживать через Kapacitor, так как может такого и нельзя :)
4. Каналы алертинга. Заявлена поддержка HipChat, OpsGenie, Alerta, Sensu, PagerDuty, Slack, VictorOps, кроме этого есть запись в лог, email, POST-запрос. Для разных событий можно указывать разные каналы. Monit умеет только email, а мне нужен был Slack.

### Плюсы Monit:
1. Monit проверенный, а Kapacitor - нет, как и весь TICK.
2. Monit имеет прямой доступ к серверу, что позволяет ему реагировать самостоятельно, например, перезагружать сервис, если он не отвечает. Kapacitor умеет только уведомлять.


## Установка
Ставить можно [по-разному](https://influxdata.com/downloads/#kapacitor).

Для тех, кто не дружит с Ansible, установка из репозитория, [взятая из мануала](https://docs.influxdata.com/influxdb/v0.13/introduction/installation/) по InfluxDB (репозиторий один на весь стек InfluxData):

``` sh
curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
aptitude update
aptitude install kapacitor
```

Я буду ставить через Ansible [rossmcdonald/kapacitor](https://github.com/rossmcdonald/kapacitor)

``` sh
ansible-galaxy install rossmcdonald.kapacitor
ansible-playbook -c local kapacitor.yml
```

#### <a name="github-pull-request"></a>Как просто делать Pull request
В плейбуке была ошибка, я бы об этом не упоминал, если бы не узнал недавно, как просто [делать pull request](https://github.com/rossmcdonald/kapacitor/pull/1) прямо в браузере. Это заняло минуты две: жмем "редактировать" на интересующем файле, правим, ниже пишем сообщение к коммиту, сохраняем. Это автоматом создаст форк, отдельную ветку и сделает туда коммит. На следующей странице останется нажать "Create pull request".



## Настройка
Так как я уже использовал готовую ansible-роль, настройка уже включена в установку. Я взял [тестовый плейбук](https://github.com/rossmcdonald/kapacitor/blob/master/test.yml) роли и изменил его: добавил данные авторизации в InfluxDB, SMTP, Slack. Опция `global` в настройках канала для уведомлений означает, что он будет использоваться по умолчанию в скриптах, иначе его нужно указывать явно.


Для установки сделал такой плейбук:

``` yaml kapacitor.yml
- hosts: all
  roles:
    - role: rossmcdonald.kapacitor
  vars:
    # [influxdb]
    kapacitor_influxdb_enabled: "true"
    kapacitor_influxdb_urls:
      - http://localhost:8086
    kapacitor_influxdb_username: user
    kapacitor_influxdb_password: pass

    # [smtp]
    kapacitor_smtp_enabled: "true"
    kapacitor_smtp_host: smtp.yandex.ru
    kapacitor_smtp_port: 587
    kapacitor_smtp_username: example@yandex.ru
    kapacitor_smtp_password: pass
    kapacitor_smtp_from: example@yandex.ru
    kapacitor_smtp_to:
      - "admin@example.com"

    # [slack]
    kapacitor_slack_enabled: "true"
    kapacitor_slack_url:  https://hooks.slack.com/services/G2JFW7VFQ/B13UHEN5X/9J6IVIcUw9FGCeF7hfjFNGBn # url ненастоящий
    kapacitor_slack_channel: "#servers"
    kapacitor_slack_global: "true"

    kapacitor_tasks_to_enable:
```


## Проверка
Лучший способ проверить, что Kapacitor видит данные из InfluxDB - записать фрагмент:

``` sh
kapacitor record stream -name la_alert -duration 5s
```

Если запись пошла, можно приступать к самому интересному: созданию алертов.

Если через 5 секунд команда не завершилась, значит что-то пошло не так.
Смотрим логи:

- Kapacitor может говорить об ошибках к подключению к InfluxDB
- InfluxDB может сыпать `connection refused` ошибками

В моем случае домен, который я прописал в конфиге Kapacitor, был прописан в /etc/hosts на 127.0.1.1, Kapacitor слушал этот порт, соответственно, InfluxDB не мог достучаться из Docker-контейнера.


#### Проблема из-за Docker
У меня в логах была ошибка:

```
open server: open service *influxdb.Service: subscription already exists
```

Я указал другой локальный хост, localhost, т.к. я не предполагаю, что к kapacitor будет обращаться кто-то, кроме InfluxDB, который стоит на той же машине. Это не помогло. Я не понял, в чем ошибка, nmap показывает свободный порт. Оставил стандартный, поддомен машины, это почему-то сработало.

Оказалось, проблема была в том, что InfluxDB при первом запуске Kapacitor'а создал на него подписки (subscriptions), которые означают то, что InfluxDB будет пересылать в Kapacitor все, что приходит в него.

InfluxDB у меня крутится в Docker'е с проброшенными портами, а Kapacitor - нет, то есть они технически были не на одной машине. Точнее, для Kapacitor'а казалось, что InfluxDB на этой же машине, но для Influx'a он на другой машине! Оказалось, что изнутри докера внутренний адрес, на который создались подписки, вел не туда, поэтому данные не доходили до Kapacitor, чтобы исправить это, понадобилось удалить подписки, узнав их имена:

``` sql
SHOW SUBSCRIPTIONS
DROP SUBSCRIPTION "kapacitor-42d050d7-5e60-462f-b079-3f8157ec2eff" ON "telegraf"."default"
DROP SUBSCRIPTION "kapacitor-42d050d7-5e60-462f-b079-3f8157ec2eff" ON "_internal"."monitor"
```


## Выводы
1. Использование Docker для InfluxDB сильно усложнило мне процесс установки при том, что ничего мне не дало: InfluxDB - это один бинарник, если у вас вся инфраструктура живет не в контейнерах, используйте установку из репозиториев, это проще. С другой стороны откатиться на предыдущую версию будет сложнее...
2. Kapacitor сильно превосходит Monit по возможностям алертинга, но уступает ему в контроле над ситуацией. Хотя можно себе представить сценарий, что Kapacitor отправляет POST-запрос с инструкциями к действиям сервису, который делает что-то, но меня такой самопальный RPC пугает.
3. Все это достаточно сырое в том смысле, что нет достаточной обвязки (оф. [контейнер для InfluxDB](https://hub.docker.com/r/library/influxdb/) появился только 16 мая, самый популярный плейбук для Kapacitor понадобилось править, чтобы установить), информации очень мало, кроме GitHub issues и документации на данный момент нет ничего. Поэтому появляющиеся проблемы решать будет сложнее.


## Ссылки
- [страница Kapacitor](https://influxdata.com/time-series-platform/kapacitor/)
- [оф. туториал](https://influxdata.com/get-started/configuring-alerts-with-kapacitor/)
- [docs](https://docs.influxdata.com/kapacitor/v0.12/)
- [influxdata/kapacitor](https://github.com/influxdata/kapacitor)
- [influxdata/kapacitor-docker](https://github.com/influxdata/kapacitor-docker)
- [ansible-role-kapacitor](https://github.com/rossmcdonald/kapacitor)
