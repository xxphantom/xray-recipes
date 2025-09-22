# Установка свежей версии Xray на OpenWrt (c целью доступа к локальной сети роутера и доступа к Интернет через вашего домашнего провайдера), основная инструкция [тут](https://github.com/xxphantom/xray-recipes/blob/main/openWRT/xray-bridge.md). 

## Предварительные требования

- Устройство с установленным OpenWrt (Проверил на 24.10)
- Доступ к интернету на устройстве
- Root-доступ к маршрутизатору
- Минимум 32 МБ свободного места в памяти

## Установка

### 1. Подготовка системы

Обновите список пакетов и установите необходимые зависимости:

```bash
opkg update
opkg install unzip wget-ssl ca-bundle
```

### 2. Определение архитектуры

Определите архитектуру вашего устройства:

```bash
uname -m
```

**Примеры соответствия:**
- `aarch64` → выберите `linux-arm64-v8a`
- `armv7l` → выберите `linux-arm32-v7a`
- `mips` → выберите `linux-mips32`

### 3. Скачивание и установка Xray

```bash
# Установите актуальную версию (проверьте на https://github.com/XTLS/Xray-core/releases)
XRAY_VERSION="v25.9.11"  # Замените на актуальную версию

# Скачайте для ARM64 (измените архитектуру в ссылке при необходимости)
wget "https://github.com/XTLS/Xray-core/releases/download/${XRAY_VERSION}/Xray-linux-arm64-v8a.zip" -P /tmp

# Распакуйте архив
unzip -d /tmp/xray-core /tmp/Xray-linux-arm64-v8a.zip
# Удалите архив
rm /tmp/Xray-linux-arm64-v8a.zip

# Установите исполняемый файл
mv /tmp/xray-core/xray /usr/bin/
chmod +x /usr/bin/xray


# Копирование баз роутинга
mkdir -p /usr/share/xray/

# Копирование МИНИМАЛЬНЫХ баз роутинга (если роутинг не нужен и мало места на флешке роутера)
wget -O /usr/share/xray/geosite.dat "https://github.com/xxphantom/xray-recipes/raw/refs/heads/main/openWRT/geodata/geosite.dat"
wget -O /usr/share/xray/geoip.dat "https://github.com/xxphantom/xray-recipes/raw/refs/heads/main/openWRT/geodata/geoip.dat"

# ИЛИ

# Копирование ПОЛНЫХ баз роутинга (если НУЖЕН роутинг и ЕСТЬ место на флешке роутера)
mv /tmp/xray-core/geosite.dat /usr/share/xray/
mv /tmp/xray-core/geoip.dat /usr/share/xray/

# Копирование конфигурации
scp ./config.json root@192.168.1.1:/etc/xray
# Очистите временные файлы
rm -rf /tmp/xray-core
```

### 4. Проверка установки

```bash
xray version
```

Вы должны увидеть информацию о версии Xray.

## Настройка автозапуска

### 1. Создание init-скрипта

```bash
cat > /etc/init.d/xray << 'EOF'
#!/bin/sh /etc/rc.common

START=99
STOP=15
USE_PROCD=1

PROG=/usr/bin/xray
CONF=/etc/xray/config.json

start_service() {
    # Проверяем существование конфигурационного файла
    [ -f "$CONF" ] || {
        echo "Configuration file $CONF not found!"
        return 1
    }
    
    # Создаем директорию для логов
    mkdir -p /var/log/xray
    
    procd_open_instance
    procd_set_param command $PROG run -config $CONF
    procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_set_param pidfile /var/run/xray.pid
    procd_close_instance
}

stop_service() {
    killall xray 2>/dev/null
}

reload_service() {
    stop
    start
}
EOF

# Установите права доступа и включите автозапуск
chmod +x /etc/init.d/xray
/etc/init.d/xray enable
```

### 2. Создание директории конфигурации

```bash
mkdir -p /etc/xray
```

### 3. Создание конфигурации

```bash
cat > /etc/xray/config.json << 'EOF'
Здесь должен быть ваш конфиг файл Xray
EOF
```

## Управление сервисом

### Запуск и остановка

```bash
# Запуск
/etc/init.d/xray start

# Остановка  
/etc/init.d/xray stop

# Перезапуск
/etc/init.d/xray restart

# Проверка статуса
/etc/init.d/xray status
```

### Проверка работы

```bash
# Проверка процесса
ps | grep xray

# Проверка логов
tail -f /var/log/xray/error.log
```

### Тестирование конфигурации

```bash
xray test -config /etc/xray/config.json
```

## Обновление

Для обновления Xray повторите шаги 3-4 из раздела установки с новой версией:

```bash
# Остановите сервис
/etc/init.d/xray stop

# Повторите установку с новой версией
XRAY_VERSION="v25.9.11"  # Новая версия
# ... повторите шаги установки

# Запустите сервис
/etc/init.d/xray start
```

## Устранение неполадок

### Проблемы с запуском

1. **Проверьте права доступа:**
   ```bash
   ls -la /usr/bin/xray
   chmod +x /usr/bin/xray
   ```

2. **Проверьте конфигурацию:**
   ```bash
   xray test -config /etc/xray/config.json
   ```

3. **Проверьте логи:**
   ```bash
   logread | grep xray
   tail -20 /var/log/xray/error.log
   ```

## Безопасность

1. **Ограничьте доступ к конфигурации:**
   ```bash
   chmod 600 /etc/xray/config.json
   ```
