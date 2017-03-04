---
layout: post
title: "Bash on Ubuntu on Windows: первый блин"
date: '2016-04-11 05:01'
categories:
  - bash
  - windows
  - ubuntu
  - microsoft
---

Итак, [дождался обновления](http://blog.popstas.ru/blog/2016/04/07/windows-ubuntu-bash-insider-update-not-available/) Windows, поставил в нее Ubuntu [по инструкции](http://blog.zacorp.ru/main/kak-vklyuchit-podderzhku-ubuntu-v-windows-10/), вот что было дальше:

Tl;dr: оно очень сырое, не работает почти ничего.

{% img /images/2016-04/windows-ubuntu-bash.png %}

<!--more -->
Первым делом захотелось родной zsh, берем aptitude, ставим, Ubuntu же!
Шелл открылся под root, так что sudo не нужен.

```
$ aptitude install zsh
```

Конечно, ничего не вышло :) Во-первых, aptitude не нашел файл /var/lock/aptitude,
нет проблем, ставим через `apt-get`, но оказывается, что нет инета.

Про это есть [issue#14](https://github.com/Microsoft/CommandLine-Documentation/issues/14) (а багов за 4 дня открыли 40+), оказалось, дело в DNS, лечится так:

```
$ echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

При этом не заработает ifconfig, ping, nslookup, но пакеты начнут ставиться.
apt-get при установке поругивается, но ставит.


# Zsh
```
$ apt-get update && apt-get install git zsh
```

Ок, сработало, ставлю свой [zsh-config](https://github.com/popstas/zsh-config)
Что-то пошло не так с пайпами, но в итоге он поставился. Кстати git работает как родной.

Открываю новый терминал, открывается bash, смотрю /etc/passwd, там написано, что
шелл /bin/zsh. Ладно, запускаю zsh вручную, он вывалил кучу ошибок про powerline,
что-то от zsh, никакой красоты не появилось.

Ок, упрощаем, удаляю свой конфиг, открываю чистый zsh - все равно облом.

Ладно, не в zsh счастье (или все-таки в нем?).

Открываю `mc`, он как бы работает, но после первого нажатия Enter курсоры перестают бегать.
Выходим, идем дальше.


# Python

```
$ apt-get install python-pip python-dev
```

Все поставилось.
Смотрим pip:

- `glances` - не работает
- [percol](http://blog.popstas.ru/blog/2015/12/10/interactive-bash-history-with-search/) - работает!
- `ansible` - ругается при запуске про `Function not implemented`
- `ps_mem` - конечно нет
- `httpie` - работает!


# SSH
Тащим ключ с домашней машины

```
$ rsync popstas@home:/Users/popstas/.ssh/id_dsa ~/.ssh
```

Работает!

Подключаюсь к удаленному хосту - тоже работает!
Там зашел в `mc`, стало понятно, что глючит терминал: на удаленке курсоры тоже бегают плохо.
Ок, терминал будет, потом.


# PHP

```
$ apt-get install php5-cli
```

PHP работает.
Composer ставится, но при попытке установить им что-нибудь зависает.


# Nginx
Ставится, но не стартует, в error.log пишет, что не может прибиндиться к сокету.


# Вывод
Пользоваться этим сейчас конечно нельзя и в ближайший месяц думаю можно не надеяться.
Я рассчитывал на большее, ну ладно, будем надеяться, что у Microsoft получится сделать
полноценный линукс, хотя видно, что работы тут еще немеряно.
