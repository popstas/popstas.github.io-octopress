---
layout: post
title: "average: измерение среднего времени выполнения команды в bash"
date: 2016-02-29 03:46:56 +0500
comments: true
categories: 
- bash

---

Периодически хочется посчитать среднее время, у меня были такие сценарии:

- простая проверка скорости загрузки страницы
- подбор оптимальных параметров к команде
- сравнение разных команд

Раньше я просто запускал несколько раз с `time`, смотрел результат, у уме делил.
Но мне это надоело, поэтому написал скрипт `average`.

<!-- more -->

Код лежит здесь - https://github.com/popstas/server-scripts/blob/master/bin/average

Поставить можно так:
```
curl -s "https://raw.githubusercontent.com/popstas/server-scripts/master/bin/average" > /usr/local/bin/average
chmod +x /usr/local/bin/average
```

Использовать можно так:

Запуск команды по умолчанию, 5 циклов:
```
average 'command'
```

Запуск команды с указанным кол-вом циклов:
```
average 10 'command'
```

Запуск команды с передачей кол-ва циклов через переменную окружения:
```
export CYCLES=5
average 'command'
```

Чтобы не показывать вывод команды, можно обрезать его через tail:
```
average 'command' | tail -n1
```



### Узнать среднее время загрузки страницы:
С учетом кэша:
```
average 'curl -s "http://example.com/" > /dev/null'
```

В обход кэша:
```
average 'curl -s "http://example.com/?t=$RANDOM" > /dev/null'
```

---

### Продвинутое использование
Мне надо было узнать оптимальное количество параллельных процессов для запуска тестов,
теперь я могу узнать это запуском одной команды:
```
for p in {1..10}; do echo "$p" - $(average "vendor/bin/paratest -p $p" | tail -n1); done
```
Команда переберет кол-во процессов от 1 до 10, по каждой итерации выведет среднее время.

После запуска получил такие результаты:
```
$ for p in {1..10}; do echo "$p" - $(average "vendor/bin/paratest -p $p" | tail -n1); done

1 - 1 loops, best of 5: 13.9 sec per loop
2 - 1 loops, best of 5: 7.51 sec per loop
3 - 1 loops, best of 5: 5.51 sec per loop
4 - 1 loops, best of 5: 4.51 sec per loop
5 - 1 loops, best of 5: 4.42 sec per loop
6 - 1 loops, best of 5: 4.71 sec per loop
7 - 1 loops, best of 5: 4.21 sec per loop
8 - 1 loops, best of 5: 4.23 sec per loop
9 - 1 loops, best of 5: 4.13 sec per loop
10 - 1 loops, best of 5: 4.33 sec per loop
```

Видно, что после 4 потоков разницы почти нет, а вот комп от запуска кучи параллельных процессов тормозит
очень даже заметно.

Конечно, в этом случае много ума не надо, чтобы понять, что кол-во процессов должно быть по кол-ву ядер, но я что-то засомневался :)

