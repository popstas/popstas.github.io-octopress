---
layout: post
title: "Автоматическое скачивание торрентов с Weburg в Transmission и статистика на InfluxDB & Grafana"
date: 2016-01-17 08:22:25 +0500
comments: true
categories:
- transmission-cli
- php
- influxdb
- influxdata
- grafana
- travis
- ci
- symfony
- docker
- box
- phar
- phpcs
- phpmd
- composer
- coveralls

---

У моего интернет-провайдера Планета есть бонусная программа поощрения раздачи торрентов с [weburg.net](http://weburg.net), дающая бонусы, 
их можно тратить на абонентскую плату. У меня комп постоянно включен, я сразу стал участвовать.

Поддержку раздач можно разбить на несколько задач:

1. периодически скачивать новинки фильмов
2. скачивать новые серии популярных сериалов
3. удалять то, что плохо раздается

Через пару месяцев мне это надоело, задумался об автоматизации этого процесса и вот в новогодние каникулы родился 
[transmission-cli](https://github.com/popstas/transmission-cli) - консольная утилита, решающая часть этих задач.

<iframe src="https://ghbtns.com/github-btn.html?user=popstas&repo=transmission-cli&type=star&count=true&size=large" frameborder="0" scrolling="0" width="160px" height="30px"></iframe>
[![Build Status](https://travis-ci.org/popstas/transmission-cli.svg?branch=master)](https://travis-ci.org/popstas/transmission-cli)
[![Coverage Status](https://coveralls.io/repos/popstas/transmission-cli/badge.svg?branch=master&service=github)](https://coveralls.io/github/popstas/transmission-cli?branch=master)

{% img https://github.com/popstas/transmission-cli/raw/master/doc/img/grafana.png?raw=true %}

<!-- more -->

## Возможности
- скачивание популярных торрентов с http://weburg.net
- удаление дублирующихся раздач (для сериалов)
- отправка метрик в InfluxDB (для слежения за популярностью)

# Установка
Установить клиент можно так:
``` bash
latest_phar=$(curl -s https://api.github.com/repos/popstas/transmission-cli/releases/latest | grep 'browser_' | cut -d\" -f4)
wget -O /usr/local/bin/transmission-cli "$latest_phar"
chmod +x /usr/local/bin/transmission-cli
```

Пользоваться графиками можно с трудом, потому что InfluxDB и Grafana вам придется устанавливать самостоятельно.
Я ставил то и другое в docker на свою виртуалку и пробрасывал порты на localhost, 
сейчас localhost вшит в [конфиг](https://github.com/popstas/transmission-cli/blob/master/src/Config.php), 
который по сути сейчас находится в коде.

Поставить можно так, заменив папки `/Users/popstas/lib/grafana` и `/var/lib/influxdb` на ваши, 
это укажет, где будут храниться данные InfluxDB и Grafana:
```
docker run -d \ -p 3000:3000 \
           -v /Users/popstas/lib/grafana:/var/lib/grafana \
            --name grafana grafana/grafana

docker run -d -p 8083:8083 -p 8086:8086 \
           -v /var/lib/influxdb:/data \
           --name influxdb tutum/influxdb
```

Папку от InfluxDB я оставил в виртуалке, т.к. оказалось, что InfluxDB не может работать с папкой, смонтированной в
VirtualBox из Mac OS (какой-то старый глюк docker).

Чтобы собиралась статистика, нужно добавить в cron задания, я собираю с 2 компов, поэтому добавляю 2 раза.

Также, чтобы не было конфликтов, статистика не будет отсылаться, если найдет раздачи с одинаковыми названиями,
которые обычно остаются от сериалов. Поэтому их нужно чистить перед отпправкой статистики.

Раздачи у меня скачиваются в папку, за которой следят оба Transmission, как только туда попадает торрент, раздача
сразу начинается (можно сделать, чтобы спрашивала разрешение, настраивается в Transmission).

В итоге у меня получился такой cron:
```
PATH="$PATH:/usr/local/bin"
59 * * * * transmission-cli remove-duplicates --host=localhost
59 * * * * transmission-cli remove-duplicates --host=wrtnsq
0  * * * * transmission-cli send-metrics --host=localhost
0  * * * * transmission-cli send-metrics --host=wrtnsq
1  2 * * * transmission-cli download-weburg --dest=/Volumes/media/_planeta/_torrents
```



## Результаты статистики
Никогда не знал о своих раздачах ничего, кроме рейтинга и объема розданного за все время.
Графики показали интересные вещи (о которых можно было и так догадаться):

- с 18 до 22 пик раздач, с 22 до 2 спад, с 2 до 9 все спят
- в праздники и выходные больше качают днем и до ночи, но после 2 все равно все спят
- популярные фильмы популярны обычно не больше недели
- есть популярные фильмы, которые популярны и через несколько месяцев, например "Интерстеллар"

Сейчас я могу выбрать в Grafana период в 7 дней, отсортировать раздачи по розданным Гб и получить список
раздач-кандидатов на удаление.

Со статистикой еще надо работать, что еще хочется сделать:

- нормальную группировку по периодам, сейчас группируется только за час или за весь выбранный период,
  нельзя выбрать последнюю неделю и посмотреть посуточные метрики. Я скидываю метрики и сначала не понимал,
  почему так, но тут как раз вышла статья 
  [Почему расчет перцентилей работает не так как вы ожидаете?](http://habrahabr.ru/post/274303/) и многое мне объяснила.

- добавить в метрики инфу о весе раздач и вывести эффективность раздач: например, фильм в 1080p весом в 10 Гб
  скачали на 50 Гб за неделю, а 2 Гб фильм низкого качества скачали на 10 Гб, если не учитывать вес раздач, то выходит,
  что первая раздача в 5 раз эффективнее, но если учитвать, то оказвается, что они равны.

## Техническая часть:
- Symfony console - каркас консольной утилиты
- InfluxDB - хранилище метрик
- Grafana - рисование графиков
- Composer - управление зависимостями
- Box - [сборка PHAR](http://habrahabr.ru/post/274745/)
- PHPCS, PHPMD - линтеры PHP
- Travis CI - публицация PHAR на Github
- Coveralls - сервис слежения за покрытием кода тестами

Половину из этого я ни разу не использовал, вторую половину - немного. Поэтому граблей хватает.

### Symfony console
Тут мне сказать особо нечего, фреймворки я только начинаю осваивать, пока ничего не понятно с Dependency Injection,
чувствую, что у меня переменные в функции местами прокидываются криво, а местами не прокидываются, где стоило бы.

Не понятно, как тестить через PHPUnit, как мокать объекты.

Пока радуюсь, что освоился с namespaces и использовал на практике PSR-2 и PSR-4.

Почти все идеи взяты из исходников 
[composer](https://github.com/composer/composer) и 
[transmission-api](https://github.com/MartialGeek/transmission-api)

### InfluxDB
InfluxDB не может работать с папкой, смонтированной в VirtualBox из Mac OS (какой-то старый глюк docker).

InfluxDB я раньше не видел, хотел посмотреть ее как замену для хранилища Whisper из стека
Diamond -> Carbon -> Whisper -> Graphite -> Grafana для рисования графиков сервера.

Компания, стоящая за InfluxDB с недавнего времени назвается InfluxData и предлагает свой стек 
[TICK](https://influxdata.com/time-series-platform/), в который
входит еще и алертинг по отклонениям метрик. Могу сказать о нем то, что Telegraf работает, InfluxDB работает без тормозов,
собирая с моего компа метрики раз в 10 секунд, Chronograf какой-то неполноценный, по сравнению с Grafana, 
а Kapacitor я еще не смотрел.

### Grafana
В Grafana 2.6 появилось много нового, по сравнению с 2.0, которую я видел в августе. А вообще, если кто использовал
Cacti или Graphite и не видел Grafana, посмотрите, красота неописуемая.

### Composer
Некоторые dev-пакеты (phpunit) потребовали php 5.6 для запуска, поэтому поставил 5.6 минимальной необходимой версией,
хотя по факту клиент может работать и на 5.5, а на 5.4 уже не может.

### Box
Если собирать PHAR, используя box, установленный через composer, в архив попадает много ненужных dev-пакетов.
Сначала я пытался бороться с этим исключением пакетов через box.json, потом понял, что это бесполезно 
(все пакеты не исключишь, а однажды исключишь нужный), в итоге пришел к такой схеме:

- ставим пакеты через `composer install --no-dev`
- качаем box.phar
- собираем transmission-cli.phar
- доставляем пакеты через composer update

Это в 3 раза уменьшило вес собранного архива.

### PHPCS, PHPMD
PHP Code Sniffer умеет анализировать ваш код на соответствие определенным стандартам, в моем случае PSR-2,
ставится через Composer, используется так:
```
./vendor/bin/phpcs --standard=psr2 ./src
```

А PHP Mess Detector у меня не запустился.

### Travis CI
Впервые удалось использовать его по назначению. Как-то пробовал использовать его для тестов пакета bash скриптов
[drupal-scripts](https://github.com/popstas/drupal-scripts), но быстро сдался, т.к. в окружении travis они вели себя не так,
как на локалке (в итоге перекинул тесты на TeamCity).

На этом проекте travis прогоняет тесты phpunit 
(тестов по сути еще нет, но без phpunit в каком-либо виде travis по умолчанию фейлит сборку) 
и если к коммиту был проставлен git tag,
публикует PHAR как приложение к релизу на Github, чуть подробнее я написал 
в [этом комменте](http://habrahabr.ru/post/274745/#comment_8736379).

### Coveralls
До покрытия тестами я еще не добирался, я тесты-то еще только начинаю использовать, решил попробовать на этом проекте.

Чтобы добавить coveralls в самом простом случае (в доках есть и сложные), достаточно сделать так, чтобы PHPUnit
генерил файл `build/logs/clover.xml`, для этого надо добавить строчку в phpunit.xml, в секцию logging:
``` xml
<logging>
    <log type="coverage-clover" target="build/logs/clover.xml"/>
</logging>
```
Ну и конечно зарегаться на https://coveralls.io/ и активировать там проект.
Если путь будет другой, придется читать доки и создавать файл настройки .coveralls.yml

В результате я имею красивую красную ачивку на странице проекта 
и [историю деградации покрытия](https://coveralls.io/github/popstas/transmission-cli) :)