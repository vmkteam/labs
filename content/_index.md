+++
date = '2025-05-14T10:26:46+03:00'
draft = false
title = 'vmkteam labs'
+++


#  Добро пожаловать!

**`vmkteam labs`** – ресурс, посвященный программированию на Go и около.

У нас есть Github, куда мы выкладываем проекты https://github.com/vmkteam/ и [философия](/development/philosophy/), которую мы используем при разработке.

## Простая архитектура
В разделе [про архитектуру](/simple-architecture/) вы найдете описание "простой архитектуры", которая подойдет многим (исключая бигтех).

## Работа с API
В разделе [про API](/development/mastering-api/) вы найдете хорошие практики.

## Onboarding
В данном разделе мы попробуем пройти создание проекта с чистого листа и его модификации в будущем. Познакомимся с нашими инструментами.

## Тулинг

### JSON-RPC 2.0
* [zenrpc](https://github.com/vmkteam/zenrpc) – JSON-RPC 2.0 сервер через `go generate`
  * [zenrpc-middleware](https://github.com/vmkteam/zenrpc-middleware) – полезные мидлвари для zenrpc
  * [rpcgen](https://github.com/vmkteam/rpcgen) – генератор клиентов на различных языках (Go/Dart/PHP/TypeScript/Swift)
  * [smdbox](https://github.com/vmkteam/smdbox) – UI для zenrpc
* [rpcdiff](https://github.com/vmkteam/rpcdiff) – дифф для CI между разными схемами [OpenRPC](https://open-rpc.org/) (через rpcgen)
* [brokersrv](https://github.com/vmkteam/brokersrv) – сервис для асинхронного взаимодействия через JSON-RPC 2.0

### Библиотеки
* [embedlog](https://github.com/vmkteam/embedlog) – обертка над slog c плюшками
* [cron](https://github.com/vmkteam/cron) - реализация крона с поддержкой middleware & ui
* [vfs](https://github.com/vmkteam/vfs) – библиотека/сервис для работы с файлами самым простым образом

### Инструменты 
* [pgmigrator](https://github.com/vmkteam/pgmigrator) – простые миграции для postgresql
* [colgen](https://github.com/vmkteam/colgen) – генератор сниппетов [и не только](/colgen/)
* [mfd-generator](https://github.com/vmkteam/mfd-generator) – генератор разного кода