# Настройка Xray Bridge на OpenWRT роутере

> Данное руководство основано на статье ["XRay (с VLESS/XTLS): проброс портов, реверс-прокси, и псевдо-VPN"](https://habr.com/ru/articles/774838/), где подробно описывается концепция и принципы работы Xray в режиме реверс-прокси. В нашем руководстве мы рассмотрим практическую реализацию bridge на роутере с OpenWRT.

## Подготовка

Для установки вам потребуются:

- Роутер с OpenWRT
- Доступ к роутеру по SSH
- Файлы:
  - config.json (конфигурация Xray)
  - geoip.dat (база данных IP-адресов)
  - geosite.dat (база данных доменов)

Микро версии гео файлов лежат в этом репо в папке geodata

## Пошаговая инструкция установки

### 1. Установка пакета

```bash
opkg update
opkg install xray-core
```

### 2. Копирование необходимых файлов

```bash
# Копирование баз данных
mkdir -p /usr/share/xray/ 
wget -O /usr/share/xray/geosite.dat "https://github.com/xxphantom/xray-recipes/raw/refs/heads/main/openWRT/geodata/geosite.dat"
wget -O /usr/share/xray/geoip.dat "https://github.com/xxphantom/xray-recipes/raw/refs/heads/main/openWRT/geodata/geoip.dat"

# Копирование конфигурации
scp ./config.json root@192.168.1.1:/etc/xray
```

### 3. Активация и запуск сервиса

```bash
# Включение сервиса
uci set xray.@xray[0].enabled='1'
uci commit xray

# Запуск сервиса
/etc/init.d/xray start
```

## Проверка работоспособности

После установки можно проверить статус сервиса:

```bash
/etc/init.d/xray status
```

Просмотр логов:

```bash
logread | grep xray
```

## Преимущества данной конфигурации

1. Безопасный удаленный доступ к локальной сети роутера через зашифрованное соединение
2. Возможность доступа к интернету через роутер для удаленных клиентов
3. Использование современного протокола VLESS с Reality для повышенной безопасности
4. Не требует белого IP на роутере

## Возможные проблемы и их решение

1. Если сервис не запускается:

```bash
# Проверка конфигурации
/usr/bin/xray -test -config /etc/xray/config.json

# Просмотр логов
logread | grep xray
```

2. Если нет доступа к порталу:

- Проверьте правильность настроек Reality (publicKey, serverName, shortId)
- Убедитесь, что порт 443 доступен

## Управление сервисом

### Остановка сервиса

```bash
# Остановка xray
/etc/init.d/xray stop

# Отключение автозапуска
uci set xray.@xray[0].enabled='0'
uci commit xray
```

### Перезапуск сервиса

```bash
/etc/init.d/xray restart

# Или более мягкий вариант перезапуска
/etc/init.d/xray reload
```

## Удаление

### 1. Остановка и отключение сервиса

```bash
/etc/init.d/xray stop
/etc/init.d/xray disable
```

### 2. Удаление пакета

```bash
opkg remove xray-core
```

### 3. Удаление конфигурационных файлов (опционально)

```bash
rm -rf /etc/xray
rm -rf /usr/share/xray
rm /etc/config/xray
```

После этих действий Xray будет полностью удален из системы. Если планируете переустановку в будущем, рекомендуется сохранить копию вашего конфигурационного файла.
