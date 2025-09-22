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

## Пошаговая инструкция установки, выберите один из вариантов

### 1. Установка Xray пакета из репозитория OpenWRT (просто, но Xray будет не самым свежим)

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
cat > /etc/xray/config.json << 'EOF'
Здесь должен быть ваш конфиг файл Xray
EOF
```
### 1. Установка Xray с помощью бинарника из github Xray (сложнее, но Xray будет самым актуальным) 

Выполнить шаги по установке [отсюда](https://github.com/xxphantom/xray-recipes/blob/main/openWRT/install-xray-openWRT.md)

# Пример конфига (на роутере)

```json
{
  "reverse": {
    "bridges": [
      {
        "tag": "bridge",
        "domain": "changeme.changeme"
      }]
    },
  "inbounds": [],
  "log": {
    "loglevel": "none"
  },
  "outbounds": [
    {
      "mux": {
        "concurrency": -1,
        "enabled": false,
        "xudpConcurrency": 8,
        "xudpProxyUDP443": ""
      },
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "reality.example.com",
            "port": 443,
            "users": [
              {
                "encryption": "none",
                "flow": "xtls-rprx-vision",
                "id": "change-me",
                "level": 8,
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "realitySettings": {
          "allowInsecure": false,
          "fingerprint": "chrome",
          "publicKey": "change-me",
          "serverName": "reality.example.com",
          "shortId": "change-me",
          "show": false,
          "spiderX": ""
        },
        "security": "reality",
        "tcpSettings": {
          "header": {
            "type": "none"
          }
        }
      },
      "tag": "proxy"
    },
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      },
      "tag": "block"
    }
  ],
    "routing": {
      "rules": [
      {
        "type": "field",
        "inboundTag": ["bridge"],
        "domain": ["full:changeme.changeme"],
        "outboundTag": "proxy"
      },
      {
        "type": "field",
        "inboundTag": ["bridge"],
        "outboundTag": "direct"
      }]
    }
}
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

- Проверьте правильность настроек Reality ()
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

### 2. Удаление пакета (для варианта установки в виде пакета OpenWRT)

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
