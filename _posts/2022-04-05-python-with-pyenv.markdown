---
layout: post
title:  "Управляем Python с помощью pyenv"
date:   2022-02-07 10:00:00 +0300
categories: python pyenv
---

Python, как и любая популярная технология, развивается основательно и быстро. Количество версий увеличивается, каждая привносит свои новшества, упрощающие жизнь разработчикам. Однако, многие однажды написанные проекты живут на уже устаревших на данный момент версиях интерпретатора, а какой-то острой необходимости их обновлять нет. Также бывают ситуации, когда проект должен успешно запускаться на нескольких минорных [версиях](https://semver.org/lang/ru/) в рамках одной мажорной. Как же быть в такой ситуации разработчику, которому нужно не только поддерживать проекты на старых версиях, например закрывать уязвимости, исправлять критические ошибки, но и создавать новые, на актуальных версиях Python? Можно поставить все необходимые версии интерпретаторов в систему и работать напрямую, и это жизнеспособный подход. Однако, существует специальный инструмент, который служит как раз для этой задачи - под одним интерфейсом управлять всем множеством версий Python и виртуальных окружений. Это инструмент называется [pyenv](https://github.com/pyenv/pyenv).

<!--more-->

<p align="center"><img src="https://files.realpython.com/media/pyenv-pyramid.d2f35a19ded9.png"></p>

Но почему все-таки просто не использовать системный интерпретатор? Как следует из названия - системный интерпретатор - идет в пакете с системой по умолчанию (Linux и macOS). И обычно это как раз [не самая последняя версия python](https://www.macrumors.com/2022/01/28/apple-removing-python-2-in-macos-12-3/), на которой работают разные системные утилиты:

```bash
$ which python
  /usr/bin/python
```

```bash
$ python -V
  Python 2.7.12
```

Добавлять и обновлять зависимости глобально в данном контексте - [сильная головная боль]({% link _posts/2021-07-08-python-dependency-management-tools.markdown %}), потому что различные системные пакеты могут перестать работать. Помимо этого, для добавления пакетов нужно вызывать команды с полномочиями суперюзера - `sudo pip install requests`, так как это системная утилита. Можно установить несколько интрепретаторов, но и тогда их использование может внести сильную путаницу - у каждого интерпретатора будет своя точка вызова:

```bash
$ which python
  /usr/bin/python
$ which python3
  /usr/local/bin/python3
$ which python3.7
  /usr/local/bin/python3.7
$ which python3.9
  /usr/local/bin/python3.9
```

В данном примере глобальный python - 2.7. Глобальный python3 может быть например 3.7. Чтобы вызвать 3.8, нужно обращаться напрямую к python3.8. Просто подменить глобальный python с 2.7 на 3.x не выйдет, это может поломать системные утилиты.

Может быть, лучше использовать пакетный менеджер, например apt/yum/brew? В конце концов, именно так установлены большинство пакетов в системе. Увы, проблемы будут те же, что и в случае с установкой интерпретаторов напрямую, потому что по умолчанию пакетные менеджеры устанавливают пакеты глобально в систему. Это приводит к загрязнению окружения разработки и куче дополнительных действий при переключении между версиями python. 

Чтобы упростить данный подход, и был разработан такой инструмент как pyenv. Но прежде чем перейти к примерам использования, давайте сначала попробуем разобраться, как он работает. pyenv является ответвлением проекта [rbenv](https://github.com/rbenv/rbenv), который был создан для управления версиями Ruby. Оба этих инструмента оперируют такими понятием как [shim (перехватчик, прокладка)](https://github.com/rbenv/rbenv#understanding-shims).

Для понимания работы pyenv и shim сначала нужно разобраться с тем, что такое [PATH](https://github.com/rbenv/rbenv#understanding-path). PATH - это системная переменная окружения, содержащая в себе разделенный двоеточиям упорядоченный список директорий, в которых хранятся исполняемые файлы.

```bash
$ echo $PATH
  /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

Когда в терминале вводится команда, система пытается найти исполняемый файл по порядку указанных в PATH директорий слева направо:
1. /usr/local/bin
2. /usr/bin
3. /bin
4. /usr/sbin
5. /sbin

То есть если запустить 
```bash
$ echo $PATH
```
система сначала зайдет в `/usr/local/bin`, потом в `/usr/bin`, после чего обнаружит исполняемый файл в `/bin` и выполнит команду. Проверить, в какой директории PATH лежит исполняемый файл можно командой `whereis`

```bash
$ whereis echo
  /bin/echo
```

Теперь давайте разберемся, как же связаны PATH и pyenv. Дело в том, что при установке pyenv, нужно добавить в профиль оболочки следующий код:
```bash
if command -v pyenv 1>/dev/null 2>&1; then
  eval "$(pyenv init -)"
fi
```

```bash
$ echo $PATH
  /Users/username/.pyenv/shims:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

Как видим, pyenv изменяет PATH, добавляя в начало директорию с перехватчиками. Теперь все команды будут искать исполняемые файлы начиная с нее. Это позволяет pyenv перехватить все связанные с python команды.

```bash
$ which python
  /Users/username/.pyenv/shims/python
```

```bash
$ which pip
  /Users/username/.pyenv/shims/pip
```

Внутри всех файлов в директории /Users/username/.pyenv/shims/ один и тот же код:
```bash
% cat /Users/username/.pyenv/shims/pytest

  #!/usr/bin/env bash
  set -e
  [ -n "$PYENV_DEBUG" ] && set -x

  program="${0##*/}"

  export PYENV_ROOT="/Users/username/.pyenv"
  exec "/opt/homebrew/opt/pyenv/bin/pyenv" exec "$program" "$@"
```

Как видим, все shims - это легковесные исполняемые файлы, которые просто передают исполнение перехваченной команды напрямую в pyenv. С его кодом [можно ознакомиться здесь](https://github.com/pyenv/pyenv/blob/master/libexec/pyenv).

Таким образом, если у вас глобально (через pyenv global) установлена версия, например, python 3.8.10, то после выполнения команды `pip install package`, этот пакет появится в `~/.pyenv/versions/3.8.10/lib/python3.8/site-packages`. Если изменить глобальную версию python, то путь установки изменится соответствующим образом. Пакеты скрыты друг от друга и аккуратно упорядочены.

Давайте теперь подробнее остановимся на понятиях system, global и local в контексте pyenv. Выполним несколько команд:

```bash
$ pyenv version
  3.10.2 (set by /Users/username/.pyenv/version)
```

```bash
$ pyenv versions
  system
* 3.10.2 (set by /Users/username/.pyenv/version)
  3.10.2/envs/venv-codeyard
  3.6.15
  3.8.10
  3.9.10
  venv-codeyard
```

```bash
$ pyenv global
  3.10.2
```

```bash
$ pyenv local
  pyenv: no local version configured for this directory
```

По результам выполненных команд выше видим, что в системе глобально была назначена установленная через pyenv версия 3.10.2. То есть в каком бы месте мы не обратились сейчас к python (за исключением явно заданных через local и системных утилит) - будет использоваться именно эта версия. Как это сделано? pyenv просто считывает файл `/Users/username/.pyenv/version`:

```bash
$ pyenv global
  3.10.2
$ cat .pyenv/version
  3.10.2
$ pyenv global 3.9.10
$ cat .pyenv/version
  3.9.10
$ pyenv global
  3.9.10
```

```bash
$ echo "3.10.2" > ~/.pyenv/version
$ cat .pyenv/version
  3.10.2
$ pyenv global
  3.10.2
```

Установка локальной версии работает по такому же принципу - при выполнении pyenv local 3.9.10 в директории создается файл `.python-version` с указанием локальной версии. 
```bash
~/Projects/my_project $ pyenv global
  3.10.2
~/Projects/my_project $ python -V
  Python 3.10.2
~/Projects/my_project $ pyenv local 3.9.10
~/Projects/my_project $ cat .python-version
  3.9.10
~/Projects/my_project $ python -V
  Python 3.9.10
```

Но назначение локальной версии не избавляет нас от ситуации, когда установка пакетов в конкретной локальной директории осуществится в глобальный контекст интерпретатора. Здесь нам на помощь придет старое проверенное средство - виртуальное окружение python, тем более что pyenv отлично умеет с ним работать. У pyenv есть плагин - [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv). Работа с ним почти ничем не отличается от работы с virtualenv и venv. А еще, он может автоматически активировать виртуальное окружение, при входе в директорию проекта. Создание виртуального окружения осуществляется командой:
```bash
$ pyenv virtualenv <python_version> <environment_name>
```
Версию python можно не указывать, тогда будет использована выставленная контекстом pyenv (global/local).

Теперь мы имеем общее понимание работы pyenv. Давайте разберемся с его установкой. [Официальная документация](https://github.com/pyenv/pyenv#installation) в достаточной мере описывает этот процесс. Сначала устанавливаем [необходимые зависимости](https://github.com/pyenv/pyenv/wiki#suggested-build-environment) для сборки версий python непосредственно на машине. Пример для macOS:
```bash
$ brew install openssl readline sqlite3 xz zlib
```
Теперь устанавливаем сам pyenv через brew:
```bash
$ brew install pyenv
```
либо используем [автоматический установщик](https://github.com/pyenv/pyenv-installer):
```bash
$ curl https://pyenv.run | bash
```

После установки добавляем в профиль оболочки [код инициализации](https://github.com/pyenv/pyenv#basic-github-checkout) (пример для zsh):
```bash
$ echo 'eval "$(pyenv init --path)"' >> ~/.zprofile
$ echo 'eval "$(pyenv init -)"' >> ~/.zshrc
```

Выполняем команду `pyenv commands` и готово, можно пользоваться. Полный справочник комманд можно посмотреть [здесь - Command reference](https://github.com/pyenv/pyenv/blob/master/COMMANDS.md).


Материалы:
1. [Understanding Shims](https://github.com/pyenv/pyenv#understanding-shims)
2. [Managing Multiple Python Versions With pyenv](https://realpython.com/intro-to-pyenv/)
3. [Deep dive into how pyenv actually works by leveraging the shim design pattern](https://mungingdata.com/python/how-pyenv-works-shims/)
