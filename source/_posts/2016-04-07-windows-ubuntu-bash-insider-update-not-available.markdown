---
layout: post
title: Windows 10 build 14316 со встроенной Ubuntu и bash через программу Microsoft Insider Preview доступна не всем
date: '2016-04-07 00:26'
categories:
  - microsoft
  - bash
  - ubuntu
  - update
  - insider
---

Все конечно слышали, что Microsoft и Canonical сговорились и встроили в винду линукс. Так вот, его пока еще нельзя потрогать.

UPD 11.04.2016: сборка 14316 дошла до меня, смотрите инструкцию по настройке.

- [issue про недоступность сборки 14316](https://github.com/Microsoft/CommandLine-Documentation/issues/5)
- [Инструкция по обновлению на русском](http://blog.zacorp.ru/main/kak-vklyuchit-podderzhku-ubuntu-v-windows-10/)

{% img  http://az648995.vo.msecnd.net/win/2016/04/bash-1024x569.png   %}

<!-- more -->

Вчера утром пришло письмо от Microsoft:

> Только что закончилась ежегодная конференция разработчиков Build 2016, на которой мы представили новые функции Windows 10.
> Вы сможете в числе первых опробовать эти новые функции, выбрав "быстрый" или "медленный" круг обновлений. Подробную информацию о новых возможностях читайте в записи блога Гейба о последней сборке Windows 10 Insider Preview. Обратите внимание, что для участников программы предварительной оценки, которые хотят выполнить чистую установку этой сборки или запустить ее в виртуальной машине, доступны ISO-образы. (на английском языке.)

Я конечно пришел домой вечером, скачал ISO, поставил на Virtualbox, сегодня на обеде думал: "Приду домой, посмотрю, что там за линукс". Удивлялся, что до сих пор не гуглятся обзоры фичи.

Включил виртуалку, x64, English, что дальше делать - не понятно. Нагуглил официальные страницы фичи:

- [Анонс билда](https://blogs.windows.com/windowsexperience/2016/04/06/announcing-windows-10-insider-preview-build-14316/)
- [Видео с конференции BUILD](https://msdn.microsoft.com/en-us/commandline/wsl/about)
- [Github](https://github.com/Microsoft/CommandLine-Documentation)
- [Инструкция по включению фичи](https://github.com/Microsoft/CommandLine-Documentation/blob/master/commandline/WSL/install_guide.md)

Уже написана [инструкция по обновлению до Windows subsystem for Linux](http://blog.zacorp.ru/main/kak-vklyuchit-podderzhku-ubuntu-v-windows-10/) на русском, но до рабочей фичи ее автор тоже не дошел.

Чтобы обновление пришло, нужно в настройках центра обновлений включить режим разработчика, переключиться на Insider level: fast, обновиться минимум до build 14316

Все стало понятно из этого [issue](https://github.com/Microsoft/CommandLine-Documentation/issues/5), кто-то, включая меня, застрял на сборке 14295.

Ждем.
