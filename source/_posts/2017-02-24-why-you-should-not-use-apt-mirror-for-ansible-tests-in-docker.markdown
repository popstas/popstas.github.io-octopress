---
layout: "post"
title: "Как я создал и отказался от локального репозитория apt-mirror для Ubuntu для ускорения тестирования ansible ролей"
date: "2017-02-24 17:39"
comments: true
categories:
- ubuntu
- apt
- apt-mirror
- apt-cacher
- apt-cacher-ng
- gitlab
- ci
- docker
- ansible
- molecule
- artifactory

---

При тестировании плейбуков на чистой Ubuntu (а как же еще?) самые большие накладные расходы по времени (субъективно)
и уж точно самые большие по трафику уходят на установку пакетов из системного репозитория. Особенно это заметно, когда видишь, что один и тот же тест Travis CI прогоняет в 1.5 раза быстрее.

Ниже описано, как создать зеркало из http://mirror.yandex.ru/ubuntu и подружить его с Gitlab CI и molecule.

Tl;dr: не делайте локальный репозиторий через `apt-mirror` для мелких задач, не стоит оно того. Вместо этого нужно поднять кеширующий сервер через [apt-cacher-ng](/blog/2017/02/26/apt-cacher-ng-for-testing-ansible-roles-with-docker-and-gitlab-ci/).

<img src="/images/2017-02/apt-mirror.png" />

<!-- more -->

## Настройка apt-mirror
Для синхронизации локального репозитория с основным вариант один - `apt-mirror`.
[Официальный сайт](https://apt-mirror.github.io) считает нас умными, поэтому все его инструкции заключаются в 3 строчках:
``` bash
apt-get install apt-mirror
nano /etc/apt/mirror.list
sudo apt-mirror
```

Все действительно почти так просто. Почти.

### Выбор самого быстрого репозитория
Пока гуглил тему, случайно наткнулся на [инструкцию](https://hub.docker.com/r/evgeniyklemin/ubuntu-fastest-apt-mirror/), как выбрать самый быстрый репозиторий.
Скорее всего, для нас для всех это будет http://mirror.yandex.ru/ubuntu, но можно в этом убедиться:

``` bash
wget -q -nv -O- http://ftp.ru.debian.org/debian/pool/main/n/netselect/netselect_0.3.ds1-26_amd64.deb > /tmp/netselect_0.3.ds1-26_amd64.deb
dpkg -i /tmp/netselect_0.3.ds1-26_amd64.deb
netselect -s3 -t20 `wget -q -nv -O- https://launchpad.net/ubuntu/+archivemirrors | grep -P -B8 "statusUP|statusSIX" | grep -o -P "(f|ht)tp.*\"" | tr '"\n' '  '`
```

Пакета нет в репозитории Ubuntu, поэтому качаем из репозитория Debian
В результате вы получите список из 3 самых быстрых (по пингу) репозиториев:

```
54 http://mirror.yandex.ru/ubuntu/
89 http://ubuntu.volia.net/ubuntu-archive/
124 http://nl.archive.ubuntu.com/ubuntu/
```

### Конфигурация
Открываем `/etc/apt/mirror.list`.

- Меняем `archive.ubuntu.com` на `mirror.yandex.ru`.
- Убираем `multiverse` репозиторий (в стандартном Docker контейнере `ubuntu` его нет, видимо не очень нужен, зато экономим сразу 13 Гб).
- Меняем путь хранения зеркала, не забывая после этого скопировать пустой скрипт в новое место `/var/spool/apt-mirror/var/postmirror.sh`, иначе `apt-mirror` будет в конце падать с ошибкой. У меня зеркало будет храниться в `/var/backups/apt-mirror` (на диске с бекапами места много)

Это же в виде команд:
``` bash
sed -i /etc/apt/mirror.list 's/archive.ubuntu.com/mirror.yandex.ru/g'
sed -i /etc/apt/mirror.list 's/ multiverse//g'
sed -i /etc/apt/mirror.list 's/\/var\/spool\/apt-mirror/\var\/backups\/apt-mirror/g'
mkdir -p /var/backups/apt-mirror/var
cp /var/spool/apt-mirror/var/postmirror.sh /var/backups/apt-mirror/var
```

Добавляем в cron задание по обновлению репозитория, я буду запускать в 1 ночи:
``` bash
sed -i 's/#0 4/0 1/g' /etc/cron.d/apt-mirror
```

Настраиваем nginx на отдачу репозитория, у меня конфиг такой:
``` bash
server {
  listen 80;
  server_name mirror.myserver.ru;
  root /var/backups/apt-mirror/mirror/mirror.yandex.ru;
  access_log off;

  location / {
    autoindex on;
  }
}
```

Все готово, осталось запустить `apt-mirror` и подождать денек: у меня выкачивалось 142 Гб.
Причем обновления тоже будут весить ощутимо, как я понял: через день я запустил apt-mirror еще раз,
он скачал 1.5 Гб.

Проверяем URL http://mirror.myserver.ru/, там должен быть доступен каталог `ubuntu`.

После этого можете сменить системные репозитории в ваших локальных убунтах и наслаждаться скоростью.


### Ошибка при apt-get update: binary-i386/Packages: 404  Not Found
Хотя нет, насладиться сразу конечно не получилось. По какой-то причине (наверное причина в месте на диске), apt-mirror выкачивает только amd64 пакеты, из-за чего `apt-get update` ругается:
```
W: The repository 'http://apt.myserver.ru/ubuntu xenial-backports Release' does not have a Release file.
W: Failed to fetch http://apt.myserver.ru/ubuntu/dists/xenial/main/binary-i386/Packages: 404  Not Found
W: Failed to fetch http://apt.myserver.ru/ubuntu/dists/xenial-updates/main/binary-i386/Packages: 404  Not Found
E: Some index files failed to download. They have been ignored, or old ones used instead.
```

Казалось бы ничего страшного, но уверен, что в тестах ненулевой код выхода apt-get будет все останавливать, поэтому придется чинить.

Ошибка есть на [askubuntu.com](https://askubuntu.com/questions/465303/apt-mirror-error/574141), спасибо человеку, который предложил решение и негодовал по поводу того, что есть только в `man sources.list`.

Решение напрашивается: явно указывать в `sources.list`, что в репозитории только amd64 пакеты, то есть вместо:
```
deb [ arch=amd64 ] http://apt.myserver.ru/ubuntu/ xenial main restricted universe
```

С настройкой `apt-mirror` закончили, перейдем к использованию в тестах.

---


## Переключение Docker контейнера на локальный apt репозиторий
https://github.com/ekino/docker-images/tree/master/apt-mirror - здесь приведено 2 способа настройки репозитория в контейнере, не изменяя его:

1. [Плохой способ] Подмена через DNS
2. [Хороший способ] Подмена `/etc/apt/sources.list`

Я выбрал хороший. Делается это монтированием файла на место `/etc/apt/sources.list`:

``` bash
FQDN="apt.myserver.ru"
cat <<EOF > sources.list-$FQDN
deb [ arch=amd64 ] http://$FQDN/ubuntu/ xenial main restricted universe
deb [ arch=amd64 ] http://$FQDN/ubuntu/ xenial-updates main restricted universe
deb [ arch=amd64 ] http://$FQDN/ubuntu/ xenial-security main restricted universe
EOF
```

Чтобы не тащить с собой артефакты, файл создается командой.

После этого проверяем, это должно отработать нормально:
``` bash
docker run --rm -it -v $(readlink -f sources.list-$FQDN):/etc/apt/sources.list ubuntu:16.04 apt-get update
```

Если `readlink` выдает ошибку `readlink: illegal option -- f`, тогда вы скорее всего сидите на MacOS и вам нужно сделать `brew install coreutils` и прописать в переменную `PATH` то, что он просит.


## Сравнение скорости
Я потратил около 4 часов на то, чтобы настроить локальные репозитории, посмотрим, сколько я сэкономил времени.
Скорость инета у меня 30 мбит.

Я сравнил отработку `time molecule test` на 3 ansible ролях, вот результаты:

Роль                | Стандартный репозиторий | Локальный репозиторий | Travis CI:
--------------------|-------------------------|-----------------------|-----------
ansible-role-common | 8:04                    | 6:18                  | **4:32**
ansible-role-mysql  | 3:41                    | **3:22**              | 3:46
ansible-role-zsh    | 3:29                    | **2:54**              | 4:08

---

Как видно, прирост небольшой, всего 20-30%.
UPD 26.02.2017: на при написании [статьи про apt-cacher-ng](/blog/2017/02/26/apt-cacher-ng-for-testing-ansible-roles-with-docker-and-gitlab-ci/) я перепроверил результаты и разница сократилась до 10-20%.

Тут надо заметить, что в `test` входит проверка идемпотентности, где никакие пакеты не ставятся. Тогда я сравнил время выполнения 'molecule converge' для `ansible-role-mysql` и получил немного лучшие результаты: 2:30 против 3:17, это уже почти в 2 раза быстрее.

Роль                | Стандартный репозиторий | Локальный репозиторий
--------------------|-------------------------|----------------------
ansible-role-common | 8:15                    | 6:09
ansible-role-mysql  | 3:17                    | 2:30               
ansible-role-zsh    | 4:05                    | 2:43             



## Выводы по поводу apt-mirror
Результаты меня немного расстроили. Оказалось, что поразительного прироста в скорости, на который я надеялся, не будет.

### Плюсы:
- один раз потратил время, чтобы при каждом тесте ждать меньше
- уменьшает желание тестировать не на чистой машине
- интернет-канал не занимается в рабочее время

### Минусы
- эффект слабый, 20-30%
- сложности с пробросом файла `sources.list`
- уход от стандартной конфигурации Gitlab CI
- разные конфиги для Travis CI и Gitlab CI

На основе этого сделал для себя вывод: это подходит только для локального постоянного применения, в остальных случаях минусы перевешивают.



## Что-то тут не так...
После этого я задумался: а как делают "большие"? Из серьезных решений для локальных репозиториев я знаю только Artifactory. Пошел посмотреть, как у них обстоят дела с зеркалами и [нашел](https://www.jfrog.com/knowledge-base/how-to-mirror-a-remote-repository/): они умеют быть зеркалом, но не рекоменуют их так использовать, т.к. это неэффективно. Вместо этого они предлагают пользоваться ими как кеширующим сервером. Такие дела...

UPD 26.02.2017: перешел на использование apt-cacher-ng, в моем случае он лучше по всем параметрам, подробности читайте в продолжении
