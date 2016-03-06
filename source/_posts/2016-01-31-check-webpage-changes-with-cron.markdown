---
layout: post
title: "Проверяем изменения на странице через cron"
date: 2016-01-31 02:00:13 +0500
comments: true
categories: 
- bash

---

Сегодня хотел заказать ароматизаторов на [Baker Flavours](http://baker-flavors.blogspot.ru/), дошел до страницы заказа,
и увидел "Уважаемые заказчики! В связи с чрезвычайно большим количеством заказов, прием заказов временно прекращен.".

Ок, будем ждать, пока эта надпись не пропадет, а чтобы не проверять руками, будем делать это на автомате и ждать уведомления.

Строчка для crontab:
```
0 20 * * * curl -s http://bakerflavors.ru/formbf.htm | iconv -f windows-1251 -t utf-8 | grep "временно прекращен" > /dev/null || { echo "BF order started" | terminal-notifier && open http://bakerflavors.ru/formbf.htm }
```

Подробности под катом.

<!-- more -->

Нужно сделать так, чтобы я узнал о том, что этот текст со страницы пропадет.

Получаем содержимое страницы:
```
curl -s http://bakerflavors.ru/formbf.htm
```

Оказалось, что страница в windows-1251 кодировке и выдает `�`. Конвертируем:
```
curl -s http://bakerflavors.ru/formbf.htm | iconv -f windows-1251 -t utf-8
```

Проверяем наличие текста:
```
curl -s http://bakerflavors.ru/formbf.htm | iconv -f windows-1251 -t utf-8 | grep "временно прекращен" > /dev/null 
```

Когда текст пропадет, grep выдаст ненулевой exitcode. Добавляем действие на этот случай.

В начале я сделал как обычно делаю на сервере: отправил письмо через команду `mail`. Оказалось, что письмо уходит в спам.
Не стал с этим разбираться, вместо этого буду показывать notification. Уведомление сделал через 
[julienXX/terminal-notifier](https://github.com/julienXX/terminal-notifier) (Mac OS),
но тут опять вышел облом: уведомление нельзя показывать бесконечно, если я окажусь в это время не перед экраном, 
я об этом не узнаю. Поэтому буду еще и открывать страницу заказа в браузере. В итоге получилось вот это:
```
curl -s http://bakerflavors.ru/formbf.htm | iconv -f windows-1251 -t utf-8 | grep "временно прекращен" > /dev/null || { \
  echo "BF order started" | terminal-notifier && \
  open http://bakerflavors.ru/formbf.htm \
}
```
