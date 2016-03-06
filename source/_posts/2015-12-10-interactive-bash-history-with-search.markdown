---
layout: post
title: "Интерактивная Zsh history с поиском и скроллом, percol"
date: 2015-12-10 19:55:20 +0500
comments: true
categories: 
- bash
- zsh
---

Если кто не знает, в bash/zsh есть поиск по истории комманд, если нажать `Ctrl+R` и начать набирать 
команду, отобразится последняя команда из истории, для навигации можно использовать 
`Ctrl+R`, `Ctrl+Shift+R`. При этом видно одновременно видно только одну команду из истории.

Утилита [percol](https://github.com/mooz/percol#zsh-history-search) решает эту проблему.

{% img /images/2015-12/percol.gif %}

<!-- more -->

Собственно по ссылке выше готовый конфиг для zsh. Я немного изменил его под себя, 
чтобы использовать percol не только для поиска по истории:

``` bash zsh-percol https://github.com/popstas/zsh-config/blob/master/.zshrc
function exists { which $1 &> /dev/null }
if exists percol; then
	function percol_select_history() {
		local tac
		exists gtac && tac="gtac" || { exists tac && tac="tac" || { tac="tail -r" } }
		BUFFER=$(fc -l -n 1 | eval $tac | percol --query "$LBUFFER")
		CURSOR=$#BUFFER         # move cursor
		zle -R -c               # refresh
	}

	zle -N percol_select_history
	bindkey '^R' percol_select_history

	# percol based grep
	g() { percol --match-method regex --query="$*"; }
fi
```
Код я добавил в [свой .zshrc](https://github.com/popstas/zsh-config). Если [этот пулл реквест](https://github.com/robbyrussell/oh-my-zsh/pull/4582) примут, то данный код появится в составе 
[oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) в виде плагина `percol`.

Код полностью взят со страницы percol, от себя добавил функцию g(), она кстати конфиликтует с плагином git для oh-my-zsh,
зато теперь я могу писать что-то вроде:
``` bash
find . -type file | g
```
для интерактивного выбора результатов поиска и просто для замены grep. При этом доступен мультивыбор по `Ctrl+Space`.

Пример посложнее:
```
vim $(find -name "*.markdown" | g)
```
После запуска откроется список всех `.markdown` файлов в текущей и вложенных папках, выбранный файл сразу откроется в Vim.
Это как будто у вас появилась возможность приделывать midnight commander к результатам поиска!

Смотрите больше интересных примеров [на странице проекта](https://github.com/mooz/percol).

Надо сказать, что на github есть программы с таким же функционалом, как у percol, я об этом узнал на странице самого percol.
Там есть peco, клон percol на Go (а значит поставляется в виде одного бинарника). Мне проще через pip установить percol, так
что с аналогами не сравнивал.
