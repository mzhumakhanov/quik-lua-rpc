# quik-lua-rpc
RPC-сервис для вызова процедур из QLUA -- Lua-библиотеки торгового терминала QUIK (ARQA Technologies).

Содержание
=================

  * [Зачем?](#Зачем)
  * [Как пользоваться?](#Как-пользоваться)
    * [Установка программы](#Установка-программы)
    * [Установка зависимостей](#Установка-зависимостей)
	    * [Легко](#Легко)
	    * [Сложно](#Сложно)
    * [Запуск программы](#Запуск-программы)
  * [Схемы сообщений](#Схемы-сообщений)
  * [Примеры](#Примеры)
  * [Разработчикам](#Разработчикам)
  * [FAQ](#faq)

Зачем?
--------
Торговый терминал QUIK -- одно из немногих средств для торговли на российском рынке. Он предоставляет API в виде библиотеки QLua, написанной на Lua. Написать торговую программу, работающую с QUIK, на чём либо отличном от Lua до сих пор было не так просто (хотя и предпринимаются попытки вытащить API QLua в другие языки, например, в C# -- [QUIKSharp](https://github.com/finsight/QUIKSharp)).

Данный сервис представляет собой RPC-прокси над API библиотеки QLua. Сервис исполняется в терминале QUIK в виде Lua-скрипта и имеет прямой доступ к библиотеке QLua. Общение сторонних программ с сервисом осуществляется посредством [ZeroMQ](http://zeromq.org/) ("сокеты на стероидах"), реализуя паттерн REQ/REP (Request / Response), по протоколу TCP (к сожалению, ZeroMQ на данный момент не поддерживает Windows Named Pipes, что , вероятно, немного сократило бы транспортный оверхэд). Запросы на вызов удалённых процедур и ответы с результатами выполнения этих процедур передаются в бинарном виде, сериализованные с помощью [Protocol Buffers](https://developers.google.com/protocol-buffers/). 

Помимо вызова удалённых процедур сервис также может рассылать оповещения о событиях терминала QUIK, реализуя паттерн PUB/SUB (Publisher / Subscriber).

Соответственно, выбор языка программирования для взаимодействия с QLua ограничивается лишь наличием на этом языке реализаций ZeroMQ и Protocol Buffers, коих довольно большое количество.

Как пользоваться?
--------
### Установка программы

Скопировать репозиторий в `%PATH_TO_QUIK%/lua/`, где `%PATH_TO_QUIK%` -- путь до терминала QUIK. Если папки `lua` там нет, нужно её создать.

### Установка зависимостей

#### Легко

Распаковать архив `redist.zip`, лежащий в корне репозитория, и следовать инструкциям согласно именам папок. Если боитесь запускать приложенные .exe-файлы, то можете скачать соответствующие файлы с сайта Microsoft самостоятельно (обратите внимание, что нужны версии для платформы `x86`): https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads.

#### Сложно

<details>
	<summary>Установка <b>LuaRocks</b> (менеджер пакетов для Lua)</summary>

1. Где взять
	* Архивы с дистрибутивами: http://luarocks.github.io/luarocks/releases/
	* Инструкцию по установке можно найти здесь: https://github.com/luarocks/luarocks/wiki/Installation-instructions-for-Windows
2. Разархивировать, в командной строке Windows (cmd.exe) перейти в разархивированную папку.
3. Установить: `install.bat /NOREG /L /P %PATH_TO_LUAROCKS%`, где %PATH_TO_LUAROCKS% -- путь, куда нужно установить LuaRocks. Например, `D:/Programs/Lua/LuaRocks`.

	Почитав мануал, опции для установки можете настроить по своему вкусу.
	Например, /L значит "установить также дистрибутив Lua в папку с LuaRocks" -- он нам пригодится далее, т.к. не у всех стоит отдельный дистрибутив Lua.

	На самом деле, нам нужна не вся Lua, а её бинарники (.dll) и заголовочные файлы. Если не хотите ставить ту, что идёт с LuaRocks, то минимальный набор файлов можно взять здесь: http://luabinaries.sourceforge.net/download.html (например, `lua-5.3.4_Win32_bin.zip`). Качать нужно 32-битные версии (Win32), т.к. QUIK использует 32-битную Lua.
	
</details>

<details>
	<summary>Установка <b>protobuf-lua</b> (Lua-биндинг для Protocol Buffers -- инструмент для сериализации/десериализации)</summary>

1. Скачать Lua-биндинг для protobuf отсюда: https://github.com/Enfernuz/protobuf-lua

	Это форк форка форка :smile:, наверное, единственного Lua-биндинга для protobuf. По  мере работы с ним я внёс некоторые изменения в плагин для генерации Lua-кода, поэтому эта версия будет полезна тем, кто пожелает доработать RPC-сервис по своему усмотрению.
2. Папку `protobuf` поместить в `%PATH_TO_QUIK%/lua/`
3. Скомпилировать файл protobuf/pb.c как DLL под свою машину. 

	Для компиляции я пользовался MinGW с командной оболочкой в виде MSYS.
	1. В терминале MSYS переместиться в папку /protobuf/, где находится файл pb.c
	2. Чтобы файл скомпилировался под Windows, нужно убрать/закомментировать строчки 23-33:
		```С
		#if defined(_ALLBSD_SOURCE) || defined(__APPLE__)
		#include <machine/endian.h>
		#else
		#include <endian.h>
		#endif
		```
		Эти строчки можно убрать безболезненно, т.к. процессоры архитектуры x86 и amd64 имеют little endianness, так что препроцессор не вставит функции из endian.h, которые используются далее в файле, в конечный код.
	
	3. Получить объектный файл: `gcc -O3 -I%PATH_TO_LUA%/include -с pb.c`, где `%PATH_TO_LUA%` -- путь до дистрибутива Lua. Если ставили Lua в комплекте с LuaRocks, то это будет путь до LuaRocks. 
	
		Пример: `gcc -O3 -ID:/programs/LuaRocks/include -с pb.c`
	
	4. Получить DLL: `gcc -shared -o pb.dll pb.o -L%libraries_folder% -l%lua_library%`, где `%libraries_folder%` -- папка с .dll-библиотеками Lua, `%lua_library%` -- имя .dll-библиотеки Lua.
	
		Пример:
		* `%libraries_folder%` -- `D:/QUIK`
		* `%lua_library%` -- `qlua`
		* Итого: `gcc -shared -o pb.dll pb.o -LD:/QUIK -lqlua`

		Линковать лучше с прокси-библиотекой Lua (`qlua.dll`), которая поставляется в коробке с QUIK. Не уверен, что если слинковаться с DLL из, например, Lua for Windows, или с той, что поставляется с LuaRocks, то всё будет работать. Линковка с прокси-библиотекой lua5.1.dll, которая находится в корне QUIK, технически осуществима, но на деле при запуске скрипта происходит ошибка из-за того, что pd.dll вызовет загрузку lua5.1.dll, которая не загружается по умолчанию, и чтобы её загрузить, загрузчик начнёт рыться в системных путях. У меня в системных путях никакой lua5.1.dll не было, от того и возникала ошибка. Линковка с qlua.dll не вызывает таких проблем, т.к. эта библиотека на момент загрузки pb.dll уже загружена терминалом.
	
4. Файл pb.dll положить в `%PATH_TO_QUIK%/Include/protobuf/` , где `%PATH_TO_QUIK%` -- путь до терминала QUIK (например, `D:/QUIK`). Если папки `Include` нет, необходимо её создать.

</details>

<details>
	<summary>Установка <b>lzmq</b> (Lua-биндинг для ZeroMQ -- инструмент для межпроцессной коммуникации)</summary>
	
1. Скачать бинарники ZeroMQ для Windows
	* Страница с дистрибутивами: http://zeromq.org/distro:microsoft-windows
	* Пример дистрибутива: http://miru.hk/archive/ZeroMQ-4.0.4~miru1.0-x86.exe -- 32-битная версия, т.к. 64-битная не подойдёт.
2. Устанавливаем в `%PATH_TO_ZMQ%` -- путь выбираем произвольно, например, `D:/programs/ZeroMQ`.

	**Важно:** при установке выбрать галку `ZeroMQ headers and libraries`.
3. Переходим в `%PATH_TO_ZMQ%/lib` -- там лежат .lib-файлы от ZMQ под Windows.
	1. Определяем .lib-файл, соответствующий своей версии Windows (например, для Windows 7 это будет `libzmq-v120-mt-4_0_4.lib`).
	2. Копируем найденный .lib-файл, копию переименовываем в `libzmq.lib`.
	
	**Важно:** Не перепутайте с файлами, содержащими в имени *gd* -- это библиотеки, собранные для работы в debug-режиме.	
4. Дальше нам будет нужен компилятор от Microsoft -- MSVC. Он входит в Visual Studio. Можно скачать бесплатную MS Visual Studio Express Edition, накликать там самый минимум при установке (поддержка C/C++). Это, наверное, самый запарный по времени пункт из всех. Инструкцию не прилагаю -- надеюсь, там всё довольно просто.
5. Установка пакета `lzmq` с помощью LuaRocks
	1. Открыть Developer Command Prompt, которая поставляется с Visual Studio (можно найти через Пуск, начав искать "Command").
	2. В Developer Command Prompt перейти в `%PATH_TO_LUAROCKS%` (путь, куда установили LuaRocks)
	3. Выполнить команду: 
	
	`luarocks install lzmq ZMQ_INCDIR="%PATH_TO_ZMQ%/include" ZMQ_LIBDIR="%PATH_TO_ZMQ%/lib"`, где `%PATH_TO_ZMQ%` -- путь до установленного в п. 2 ZeroMQ.
	
	Пример: `luarocks install lzmq ZMQ_INCDIR="D:/programs/ZeroMQ 4.0.4/include" ZMQ_LIBDIR="D:/programs/ZeroMQ 4.0.4/lib"`
	
	LuaRocks начнёт устанавливать библиотеку `lzmq`, попутно собирая её из исходников. Делает он это с помощью компилятора `cl` -- за этим мы и ставили MSVC и заходили в Developer Command Prompt.
6. После установки `lzmq` заходим в `%PATH_TO_LUAROCKS%/systree/lib/lua/5.1` (путь после `%PATH_TO_LUAROCKS%` у вас может отличаться, если при установке LuaRocks вы использовали опцию `/TREE %dir%`) и копируем содержимое папку `lzmq` и файл `lzmq.dll` в `%PATH_TO_QUIK%/Include`.
7. Заходим в `%PATH_TO_LUAROCKS%/systree/share/lua/5.1` и копируем папку `lzmq` в `%PATH_TO_QUIK%/lua`
8. `lzmq.dll`, которую собрал LuaRocks, линкуется с `libzmq.lib`, которая, в свою очередь, является не полноценной статической библиотекой (разработчики ZMQ обещали предоставить такие в следующих релизах), а библиотекой импорта. Эта библиотека импорта ссылается на соответствующую ей .dll-библиотеку (например, `libzmq-v120-mt-4_0_4.dll`, если переименовывали `libzmq-v120-mt-4_0_4.lib`). Поэтому, чтобы библиотека `libzmq` была доступна в runtime, необходимо скопировать соответствующий файл (например, `libzmq-v120-mt-4_0_4.dll`) из `%PATH_TO_ZMQ%/bin` в `%PATH_TO_QUIK%`. 
9. Установите VC++ redistributable package (x86), соответствующий той версии Visual Studio, которой собирался выбранный вами libzmq-...dll. На странице http://zeromq.org/distro:microsoft-windows приводится соответствие. Если лень что-то искать и вы использовали `libzmq-v120-mt-4_0_4.dll` или `libzmq-v110_xp-mt-4_0_4.dll`, то можете распаковать архив `redist.zip`, лежащий в корне репозитория, и найти установочный пакет в папке `установить` для своей версии Windows.

</details>
	
### Запуск программы
В терминале QUIK в меню Lua-скриптов добавить скрипт `%PATH_TO_SERVICE%/service.lua`, где `%PATH_TO_SERVICE%` -- путь до папки с программой включительно (например, `D:/QUIK/lua/quik-lua-rpc`).

Адрес, по которому доступен RPC-компонент, определяется в функции `OnInit` в первом аргументе метода `QluaService:start`.

Адрес, по которому доступен компонент для рассылки событий (PUB/SUB), определяется в функции `OnInit` во втором аргументе метода `QluaService:start`.

Вы можете использовать как оба компонента, так и один из них (передавая в качестве адреса для другого компонента `nil`).
На данный момент ZeroMQ под Windows не поддерживает IPC-абстракцию (`ipc://`), поэтому для транспортного уровня остаётся (`tcp://`).

Убедитесь, что используемые вами порты открыты.

### Схемы сообщений
Схемы сообщений расположены внутри директории `qlua/rpc` в виде файлов .proto (Protocol Buffers).

### Примеры

Пример клиента на Java: https://github.com/Enfernuz/quik-lua-rpc-java-client

### Разработчикам

*COMING SOON*

### FAQ

Q: **Используешь Protocol Buffers, но не используешь gRPC. Как так?**

A: Для Lua пока не запилили генерацию стабов gRPC. Что уж говорить, даже биндинг для Protocol Buffers -- это fan-made опенсорс, оставляющий желать лучшего.

Q: **А что насчёт Thrift? Там вроде есть поддержка Lua.**

A: Если мне память не изменяет, там в зависимостях библиотеки, для которых исходники только под UNIX (например, `luabpack`).
