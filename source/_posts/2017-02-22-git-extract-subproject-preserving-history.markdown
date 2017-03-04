---
layout: "post"
title: "Извлечение одной из папок в git репозитории в отдельный репозиторий с сохранением истории - git-extract-subproject"
date: "2017-02-22 01:19"
comments: true
categories:
- bash
- git
- ansible
- server-scripts

---

Занялся я тут распиливанием большого проекта (дерево ansible ролей) на отдельные репозитории.

### Для этого надо:
1. Извлечь директорию подпроекта в отдельный репозиторий
2. Удалить из проекта папку подпроекта
3. Добавить в большой проект зависимость от подпроекта

Ниже написано, как сделать 1-й шаг одной командой через скрипт `git-extract-subproject`.

<img src="/images/2017-02/git-extract-subproject.jpg" />

<!-- more -->

В общем все оказалось просто, за минуту находится статья об этом - [
Moving Files from one Git Repository to Another, Preserving History](http://gbayer.com/development/moving-files-from-one-git-repository-to-another-preserving-history/), за 10 минут становится понятно, что как работает.

Мне нужно было проделать эту операцию 10+ раз, поэтому написал скрипт, извлекающий репозиторий одной командой.

## Алгоритм извлечения:
1. Клонировать большой проект во временный репозиторий
2. Удалить из него все, кроме папки модуля через git-фильтр. При этом переписывается история
3. Создать чистый репозиторий для нового модуля
4. Добавить в чистый репозиторий временный, как remote source
5. Сделать pull из remote source в master
6. Подчистить следы

По идее уже после п.2 временный репозиторий выглядит как готовый модуль, пп.3-6 нужны для того, чтобы не тащить следы истории и настроек родительского проекта в дочерний.

Например, у меня есть репозиторий `ansible-server`, в нем лежит роль `roles/server-scripts`. Тогда нужно перейти в папку ansible-server и запустить:

```
git-extract-subproject roles/server-scripts ansible-role-server-scripts
```

После этого рядом с `ansible-server` создастся готовый проект `ansible-role-server-scripts`. Остается добавить в него remote origin куда следует и запушить.

В итоге получился репозиторий с историей - [viasite-ansible/ansible-role-server-scripts](https://github.com/viasite-ansible/ansible-role-server-scripts/commits/master).

Код скрипта здесь - [popstas/server-scripts](https://github.com/popstas/server-scripts/blob/master/bin/git-extract-subproject)
