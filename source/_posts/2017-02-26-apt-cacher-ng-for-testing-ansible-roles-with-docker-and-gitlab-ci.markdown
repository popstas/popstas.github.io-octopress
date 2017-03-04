---
layout: "post"
title: "Настройка кеширующего прокси apt-cacher-ng для ускорения тестирования ansible ролей с Molecule, Gitlab CI и Docker"
date: "2017-02-26 04:14"
comments: true
categories:
- ubuntu
- apt
- apt-mirror
- apt-cacher
- apt-cacher-ng
- gitlab
- ci
- travis
- docker
- ansible
- molecule

---



В [предыдущей статье](/blog/2017/02/24/why-you-should-not-use-apt-mirror-for-ansible-tests-in-docker/) я настраивал `apt-mirror` для тех же целей. У того способа нашлось несколько недостатков.

В статье ниже описано, как решить ту же проблему, используя `apt-cacher-ng`.

Tl;dr: на этот раз все получилось, этот способ меня устроил.

<img src="/images/2017-02/apt-cacher-ng.png" />

<!-- more -->



## Настройка apt-cacher-ng
Здесь все довольно просто, проще, чем с `apt-mirror`.

```
apt-get install apt-cacher-ng
```

В конфиге я задал пароль админа в `/etc/apt-cacher-ng/security.conf`, он дает право смотреть подробную статистику по cache-hit.

В `/etc/apt-cacher-ng/acng.conf` интересны следующие строчки:

- `ExTreshold: 4` - устаревание кеша, в днях. Если файл ни разу не запрашивался дольше указанного времени, он будет удален. Я увеличил до 30 дней
- `PassThroughPattern: .*:443` - нужно указать это, чтобы не было проблем с HTTPS репозиториями (об этом ниже).

В остальном стандартный конфиг делает следующее:

- запускает веб-сервер для всего мира на `0.0.0.0:3142`
- хостит страничку и информацией о сервисе и статистикой на http://myserver.ru:3142
- хранит кеши в `/var/cache/apt-cacher-ng`

Также нужно отредактировать файл `/etc/apt-cacher-ng/backends_ubuntu`, удалив из него лишние зеркала и поставив главное зеркало в начало, иначе рискуете однажды получить 403 ошибку при установке одного из пакетов (об этом чуть ниже). У меня файл такой:
```
http://mirror.yandex.ru/ubuntu/
http://archive.ubuntu.com/ubuntu/
```
Подробности ремапинга можно почитать [в документации](https://www.unix-ag.uni-kl.de/~bloch/acng/html/config-serv.html). В 2 словах: когда клиент запрашивает пакет, apt-cacher-ng скачивает его не с репозитория, который прописан на клиенте, а с первого зеркала, указанного в файле ремапинга. Второй репозиторий по факту никогда не выбирается.

После этого можно перезапустить сервис:
```
service apt-cacher-ng restart
```

Проверяем, что он поднялся, должен открыться урл `http://myserver.ru:3142`.

### Ошибка 403 при получении одного из пакетов
Через некоторое время использования я споткнулся об ошибку:
```
$ apt-get install php-common -y
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
  psmisc
The following NEW packages will be installed:
  php-common psmisc
0 upgraded, 2 newly installed, 0 to remove and 7 not upgraded.
Need to get 10.8 kB/58.8 kB of archives.
After this operation, 299 kB of additional disk space will be used.
Err:1 http://archive.ubuntu.com/ubuntu xenial/main amd64 php-common all 1:35ubuntu6
  403  Forbidden
E: Failed to fetch http://archive.ubuntu.com/ubuntu/pool/main/p/php-defaults/php-common_35ubuntu6_all.deb  403  Forbidden

E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?
```

Я стал разбираться, оказалось, что дело в прокси, если его убрать, все становится нормально.

Оказалось, что репозиторий по умолчанию, который прописан в `/etc/apt-cacher-ng/backends_ubuntu.default` какой-то немного битый и пакет php-common не отдавал. Чтобы этого не произошло, нужно добавить свои репозитории в `/etc/apt-cacher-ng/backends_ubuntu`.

Solution:
```
echo http://archive.ubuntu.com/ubuntu/ > /etc/apt-cacher-ng/backends_ubuntu
service apt-cacher-ng restart
```

### Ошибка 403 при доступе к HTTPS репозиториям
В этом месте тоже появляются ошибки, проявляются в ошибках 403 при `apt-get update`.
Проблема здесь в том, что apt-cacher-ng не может прочитать зашифрованный трафик от https репозиториев, но все равно пытается. Этого можно избежать двумя способами:

- добавить такие репозитории в исключения
- использовать http репозитории в sources, а потом ремапить их на настоящие репозитории в apt-cacher-ng

Первый способ позволяет избежать изменения sources для системы-клиента apt-cacher-ng, второй - экономить трафик и для таких репозиториев. Я хочу, чтобы прокси работал максимально прозрачно, поэтому я использую первый способ. За то, какие репозитории обрабатывать, отвечает параметр `PassThroughPattern`. Нам нужно исключить из регулярного выражения все HTTPS репозитории.

Было:
```
PassThroughPattern: ^bugs.debian.org:443
```

Стало:
```
PassThroughPattern: .*:443
```

О втором способе можно прочитать в [этой статье](https://blog.packagecloud.io/eng/2015/05/05/using-apt-cacher-ng-with-ssl-tls/).



## Настройка на клиентах
На клиентах нужно добавить один файлик с указанием адреса прокси, `sources.list` менять не надо:
```
echo 'Acquire::http::Proxy "http://myserver.ru:3142";' > /etc/apt/apt.conf.d/00aptproxy
```

На хосте я этого делать не стал, т.к. у меня там стоит старая Ubuntu 14.04, а тестирую я на Ubuntu 16.04. К слову, apt-cacher-ng это не волнует, он нормально кеширует новые пакеты, не смотря на то, что стоит на старой оси. Как я понимаю, его можно использовать и в смешанном режиме, то есть кешировать пакеты сразу от нескольких версий операционок, но я это не проверял.

Вместо этого я положил файлик с указанием прокси в отдельную папку, откуда я буду пробрасывать его внутрь тестовых контейнеров:
```
echo 'Acquire::http::Proxy "http://myserver.ru:3142";' > /usr/local/src/00aptproxy
```



## Использование с Molecule, Gitlab CI и Travis CI
Не знаю зачем, но роли я тестирую сразу двумя CI: Gitlab и Travis. В связи с этим появляется проблема: нужно на Gitlab CI использовать один кеширующий сервер, при локальном тестировании другой, а для Travis CI убирать его.

Сложность в том, что Molecule не поддерживает разные конфиги, только умеет использовать в конфигах переменные окружения. Это я и использовал.

Смысл в том, что на разных CI в контейнер будут пробрасываться разные `/etc/apt/apt.conf.d/00aptproxy`, для Travis это будет просто пустой файл.

`.travis.yml`:
```
script:
  - export MOLECULE_APTPROXY_PATH="$PWD/00aptproxy"
  - touch "$MOLECULE_APTPROXY_PATH"
  - molecule --debug test
```

`molecule.yml`:
```
docker:
  containers:
    - name: ansible-role-mysql
      image: ubuntu
      image_version: latest
      volume_mounts:
        - ${MOLECULE_APTPROXY_PATH}:/etc/apt/apt.conf.d/00aptproxy
```

`.gitlab-ci.yml` я решил не менять, вместо этого я изменил способ регистрации раннеров в Gitlab CI, используются специальные раннеры с проброшенной переменной окружения:

```
gitlab-ci-multi-runner register -n \
  --executor docker \
  --description "Docker at myserver.ru on popstas/ubuntu-molecule" \
  --docker-image "popstas/ubuntu-molecule:latest" \
  --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
  --env "MOLECULE_APTPROXY_PATH=/usr/local/src/00aptproxy"
```

Это сделано потому, что я еще запускаю локальные раннеры, хотелось сделать так, чтобы `.gitlab-ci.yml` подходил во всех случаях.

На локальной машине можно просто добавить переменные окружения через `export` прямо в терминале или добавить их в ваш `~/.profile`, тогда можно просто запускать `molecule test` и все будет работать.



## Тестирование скорости

Дополню таблицу из [прошлой статьи](/blog/2017/02/24/why-you-should-not-use-apt-mirror-for-ansible-tests-in-docker/). Естественно, указано время второго прогона apt-cacher-ng для роли, т.к. в первый запуск пакеты еще не скачались, и скорость будет как при использовании стандартного репозитория.

Роль                | archive.ubuntu.org | apt-mirror | apt-cacher-ng | Travis CI:
--------------------|--------------------|------------|---------------|-----------
ansible-role-common | 8:04               | 6:18       | 6:30          | **4:32**
ansible-role-mysql  | 3:41               | **3:22**   | 3:26          | 3:46
ansible-role-zsh    | 3:16               | **2:54**   | 2:56          | 4:08

Как видим, в скорости решение с `apt-cacher-ng` по сравнению с `apt-mirror` почти не теряет. Если не видно разницы, зачем тратить лишние 140 Гб?

Кстати, скорость тестирования увеличилась и на других способах, которые я описывал в прошлой статье: если тогда разница между способами была 20-30%, то теперь она сократилась до 10-20%. Это говорит о том, что если ничего не делать и пользоваться стандартными удаленными репозиториями, вы будете больше зависеть от внешних факторов.



## Выводы

### Минусы:
- Подходит только для множественного запуска однотипных установок, в моем случае так и есть
- Немного медленнее, чем при использовании зеркала, минусом это назвать сложно, т.к. разница всего 1-3%
- Нужно пробрасывать порт через фаервол, если хотите открыть прокси всему миру, я этого делать не стал :)

### Плюсы:
- Хранит только нужные пакеты
- Кеширует не только пакеты из стандартного репозитория, но и внешние пакеты, которые вы добавляете в `sources.list`
- Не требует изменения sources.list
- Проше настраивать
- Не нужен веб-сервер (nginx)
- По умолчанию фаервол закрывает вас

Как видите, минусы надуманны, а плюсы реальны. На этом история ускорения скачивания пакетов закончена, но остается еще много интересных моментов в тестировании Ansible на Gitlab CI, продолжение следует.
