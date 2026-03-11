# Настройка OpenWRT для прокси Минцифры

Ссылка на сертификат ```https://espd.rt.ru/settings/cert-install```

Ссылка на таблицу прокси ```https://espd.rt.ru/filtering/proxy-settings```

1. Подготовка и подключение к роутеру

1.1. Подключитесь к роутеру по SSH
```bash
ssh root@192.168.1.1
```
(Если у роутера другой IP, замените 192.168.1.1 на ваш)

1.2. Узнайте ваш IP и подсеть
```bash
# Узнаем IP роутера в локальной сети
ip addr show br-lan | grep inet

# Пример вывода: inet 192.168.1.1/24 brd 192.168.1.255 scope global br-lan
# `Запомните: IP роутера и подсеть (в примере: 192.168.1.1 и 192.168.1.0/24)
```

1.3. Обновите список пакетов
```bash
opkg update
```

2. Установка необходимых пакетов
   
2.1. Установите все нужные пакеты
```bash
# Устанавливаем основные пакеты
opkg install tinyproxy \
              ca-certificates \
              ca-bundle \
              curl \
              ip-full \
              kmod-nft-tproxy \
              kmod-nft-socket \
              nftables \
              openssl-util \
              luci-app-tinyproxy  # для управления через веб-интерфейс (опционально)
```

2.2. Установка dnsmasq-full (с поддержкой nftsets)
```bash
# Проверяем, установлен ли dnsmasq
opkg list-installed | grep dnsmasq

# Если установлен dnsmasq (базовая версия), удаляем его
if opkg list-installed | grep -q "^dnsmasq "; then
    echo "Удаляем базовый dnsmasq..."
    opkg remove dnsmasq
fi

# Устанавливаем dnsmasq-full
opkg install dnsmasq-full --force-overwrite
```

2.3. Проверка установки
```bash
# Проверяем, что все пакеты установлены
opkg list-installed | grep -E "tinyproxy|kmod-nft-tproxy|nftables|dnsmasq-full|openssl-util"
```

2.4. Проверка поддержки nftset в dnsmasq
```bash
dnsmasq --version | grep nftset
# Должны увидеть: "Compiled with nftset support."
```


3. Установка корневого сертификата
   
3.1. Создайте директорию для сертификатов
```bash
mkdir -p /etc/ssl/certs/school
```

3.2. Скопируйте сертификат на роутер
На вашем компьютере (не на роутере) выполните:
```bash
# Для Linux/Mac:
scp -O /путь/к/сертификату/ca-root.crt root@192.168.1.1:/etc/ssl/certs/school/

# Для Windows (используйте WinSCP или PSCP):
# pscp ca-root.crt root@192.168.1.1:/etc/ssl/certs/school/
```

3.3. Добавьте сертификат в системное хранилище
```bash
# Делаем резервную копию
cp /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt.backup

# Добавляем сертификат
cat /etc/ssl/certs/school/ca-root.crt >> /etc/ssl/certs/ca-certificates.crt

# Проверяем, что сертификат добавился
openssl x509 -in /etc/ssl/certs/school/*.crt -text -noout | grep "Subject:"
```

4. Настройка DNS
   
После установки dnsmasq-full нужно проверить и обновить конфигурацию:

4.1. Обновление конфигурации DHCP/DNS
```bash
# Добавляем поддержку nftset в dnsmasq
# Открываем файл конфигурации
vi /etc/config/dhcp
```

Найдите секцию ```config dnsmasq``` и добавьте в неё строку:
```shell
option nftsets '1'
```

4.2. Пример полной конфигурации (если нужно восстановить)
```bash
cat > /etc/config/dhcp << 'EOF'
config dnsmasq
    option domainneeded '1'
    option boguspriv '1'
    option filterwin2k '0'
    option localise_queries '1'
    option rebind_protection '1'
    option rebind_localhost '1'
    option local '/lan/'
    option domain 'lan'
    option expandhosts '1'
    option nonegcache '0'
    option cachesize '1000'
    option authoritative '1'
    option readethers '1'
    option leasefile '/tmp/dhcp.leases'
    option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
    option nonwildcard '1'
    option localservice '1'
    option ednspacket_max '1232'
    option filter_aaaa '0'
    option filter_a '0'
    option nftsets '1'

config dhcp 'lan'
    option interface 'lan'
    option start '100'
    option limit '150'
    option leasetime '12h'
    option dhcpv4 'server'
    option dhcpv6 'server'
    option ra 'server'
    option ra_slaac '1'
    list ra_flags 'managed-config'
    list ra_flags 'other-config'

config dhcp 'wan'
    option interface 'wan'
    option ignore '1'

config odhcpd 'odhcpd'
    option maindhcp '0'
    option leasefile '/tmp/hosts/odhcpd'
    option leasetrigger '/usr/sbin/odhcpd-update'
    option loglevel '4'
EOF
```

4.3. Настройка DNS-серверов
```bash
# Добавляем внешние DNS-серверы
uci add_list dhcp.@dnsmasq[0].server='8.8.8.8'
uci add_list dhcp.@dnsmasq[0].server='8.8.4.4'
uci commit dhcp

# Перезапускаем dnsmasq
/etc/init.d/dnsmasq restart
```


5. Настройка Tinyproxy
   
5.1. Создание конфигурации Tinyproxy
```bash
cat > /etc/config/tinyproxy << 'EOF'
config tinyproxy
    option enabled 1
    option Port 8888
    option Listen 192.168.1.1
    option Timeout 600
    option LogLevel Info
    option LogFile "/var/log/tinyproxy.log"
    option User "nobody"
    option Group "nogroup"
    option MaxClients 100
    option MinSpareServers 5
    option MaxSpareServers 20
    option StartServers 10
    option MaxRequestsPerChild 0
    list Allow 192.168.1.0/24
    list Allow 127.0.0.1
    list ConnectPort 443
    list ConnectPort 563

# Прокси Минцифры для Республики Карелия (код СРФ 10)
config upstream
    option type proxy
    option via "10.0.10.52:3128"
EOF
```

ВАЖНО: Если ваш IP роутера отличается от 192.168.1.1, замените:

- option Listen 192.168.1.1 на ваш реальный IP
- list Allow 192.168.1.0/24 на вашу реальную подсеть


5.2. Запуск Tinyproxy
```bash
# Включаем автозапуск
/etc/init.d/tinyproxy enable

# Запускаем
/etc/init.d/tinyproxy start

# Проверяем, что запустился
ps | grep tinyproxy
```

6. Настройка nftables для прозрачного прокси
   
6.1. Создание правил nftables
```bash
# Создаем файл с правилами
cat > /root/tinyproxy-tproxy.nft << 'EOF'
#!/usr/sbin/nft -f

table inet tinyproxy {
    set local_ips {
        type ipv4_addr
        flags interval
        elements = {
            10.0.0.0/8,
            127.0.0.0/8,
            169.254.0.0/16,
            172.16.0.0/12,
            192.168.0.0/16,
            224.0.0.0/4,
            240.0.0.0/4
        }
    }
    
    set proxy_ports {
        type inet_service
        elements = { 80, 443 }
    }
    
    chain prerouting_tinyproxy {
        type filter hook prerouting priority mangle - 5; policy accept;
        
        meta iifname "lo" return
        ip daddr @local_ips return
        meta l4proto tcp th dport @proxy_ports meta mark set 0x1
        meta mark 0x1 meta l4proto tcp tproxy to :8888
    }
    
    chain output_tinyproxy {
        type route hook output priority mangle - 5; policy accept;
        
        ip daddr @local_ips return
        meta l4proto tcp th dport @proxy_ports meta mark set 0x1
    }
}
EOF

# Делаем файл исполняемым
chmod +x /root/tinyproxy-tproxy.nft
```

6.2. Проверка синтаксиса и загрузка правил
```bash
# Проверяем синтаксис
nft -c -f /root/tinyproxy-tproxy.nft

# Если синтаксис правильный, применяем правила
/etc/init.d/firewall restart

# Проверяем, что таблица создалась
nft list table inet tinyproxy
```

7. Создание скрипта управления

7.1. Создаем удобный скрипт для управления прокси
```bash
cat > /root/mintsifra-proxy.sh << 'EOF'
#!/bin/sh

# ======================================================
# УПРАВЛЕНИЕ ПРОКСИ МИНЦИФРЫ ДЛЯ OPENWRT
# ======================================================
# Автоматическая настройка прозрачного прокси через Tinyproxy + nftables
# Прокси-сервер Минцифры для Республики Карелия (код 10): 10.0.10.52:3128
# ======================================================

# Конфигурация - ИЗМЕНИТЕ ПОД ВАШУ СЕТЬ!
# ------------------------------------------------------
PROXY_SERVER="10.0.10.52"           # Прокси Минцифры для Карелии
PROXY_PORT="3128"                    # Порт прокси
LAN_IP="192.168.1.1"                 # IP вашего роутера в локальной сети
LAN_NET="192.168.1.0/24"             # Ваша локальная подсеть

# Системные пути
NFT_RULES_FILE="/root/tinyproxy-tproxy.nft"
TINYPROXY_CONFIG="/etc/config/tinyproxy"
NFT_TABLE_NAME="inet tinyproxy"

# Цвета для красивого вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# ======================================================
# ФУНКЦИИ ПРОВЕРКИ
# ======================================================

# Проверка доступности прокси-сервера Минцифры
check_proxy_availability() {
    if ping -c 2 -W 2 $PROXY_SERVER >/dev/null 2>&1; then
        return 0
    else
        # Пробуем проверить порт через nc если ping не работает
        if nc -zv $PROXY_SERVER $PROXY_PORT 2>&1 | grep -q "succeeded"; then
            return 0
        fi
        return 1
    fi
}

# Проверка наличия необходимых файлов
check_requirements() {
    local missing=0
    
    if [ ! -f "$NFT_RULES_FILE" ]; then
        echo -e "${RED}❌ Файл правил nftables не найден: $NFT_RULES_FILE${NC}"
        missing=1
    fi
    
    if [ ! -f "$TINYPROXY_CONFIG" ]; then
        echo -e "${RED}❌ Конфигурация Tinyproxy не найдена: $TINYPROXY_CONFIG${NC}"
        missing=1
    fi
    
    if ! command -v nft >/dev/null 2>&1; then
        echo -e "${RED}❌ nftables не установлен${NC}"
        missing=1
    fi
    
    if ! pgrep tinyproxy >/dev/null 2>&1; then
        echo -e "${YELLOW}⚠️ Tinyproxy не запущен, но будет запущен при включении${NC}"
    fi
    
    return $missing
}

# ======================================================
# ФУНКЦИИ УПРАВЛЕНИЯ NFTABLES И МАРШРУТИЗАЦИЕЙ
# ======================================================

# Включение правил nftables и маршрутизации
enable_nft_routing() {
    echo -e "${BLUE}🔧 Настройка nftables и маршрутизации...${NC}"
    
    # 1. Проверяем наличие таблицы маршрутизации
    if ! grep -q "tproxy_table" /etc/iproute2/rt_tables; then
        echo "100   tproxy_table" >> /etc/iproute2/rt_tables
        echo -e "   ${GREEN}✅ Таблица tproxy_table создана${NC}"
    fi
    
    # 2. Проверяем, загружены ли уже правила nftables
    if nft list table inet tinyproxy >/dev/null 2>&1; then
        echo -e "   ${GREEN}✅ Правила nftables уже загружены${NC}"
    else
        # Проверяем, существует ли файл с правилами
        if [ -f /root/tinyproxy-tproxy.nft ]; then
            echo -e "   Загрузка правил из /root/tinyproxy-tproxy.nft"
            nft -f /root/tinyproxy-tproxy.nft
            if [ $? -eq 0 ]; then
                echo -e "   ${GREEN}✅ Правила nftables загружены${NC}"
            else
                echo -e "   ${RED}❌ Ошибка загрузки правил nftables${NC}"
                return 1
            fi
        else
            echo -e "   ${RED}❌ Файл с правилами не найден: /root/tinyproxy-tproxy.nft${NC}"
            return 1
        fi
    fi
    
    # 3. Настройка правил маршрутизации
    # Проверяем и добавляем правило fwmark
    if ! ip rule show | grep -q "fwmark 0x1"; then
        ip -4 rule add fwmark 1 lookup tproxy_table priority 1000
        echo -e "   ${GREEN}✅ Правило fwmark добавлено в маршрутизацию${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Правило fwmark уже существует в маршрутизации${NC}"
    fi
    
    # Проверяем и добавляем маршрут через loopback
    if ! ip route show table tproxy_table | grep -q "local default dev lo"; then
        ip -4 route add local default dev lo table tproxy_table
        echo -e "   ${GREEN}✅ Маршрут через loopback добавлен в таблицу tproxy_table${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Маршрут через loopback уже существует в таблице tproxy_table${NC}"
    fi
    
    # 4. Показываем итоговую конфигурацию маршрутизации
    echo -e "\n   ${BLUE}Текущая конфигурация маршрутизации:${NC}"
    echo -e "   Правила fwmark:"
    ip rule show | grep "fwmark" | sed 's/^/      /'
    echo -e "   Таблица tproxy_table:"
    ip route show table tproxy_table | sed 's/^/      /'
    
    return 0
}

# Отключение правил nftables и маршрутизации
disable_nft_routing() {
    echo -e "${BLUE}🔧 Отключение nftables и маршрутизации...${NC}"
    
    # 1. Удаляем правила nftables (если они загружены)
    if nft list table inet tinyproxy >/dev/null 2>&1; then
        nft delete table inet tinyproxy
        echo -e "   ${GREEN}✅ Правила nftables удалены${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Правила nftables не были загружены${NC}"
    fi
    
    # 2. Удаляем правило fwmark из маршрутизации
    if ip rule show | grep -q "fwmark 0x1"; then
        # Получаем список всех правил с fwmark 0x1 и удаляем их
        RULE_IDS=$(ip rule show | grep "fwmark 0x1" | awk '{print $1}' | sed 's/://g')
        for id in $RULE_IDS; do
            ip rule del pref $id 2>/dev/null
        done
        echo -e "   ${GREEN}✅ Правила fwmark удалены из маршрутизации${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Правила fwmark не найдены в маршрутизации${NC}"
    fi
    
    # 3. Удаляем маршрут через loopback из таблицы tproxy_table
    if ip route show table tproxy_table | grep -q "local default dev lo"; then
        ip -4 route del local default dev lo table tproxy_table 2>/dev/null
        echo -e "   ${GREEN}✅ Маршрут через loopback удален из таблицы tproxy_table${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Маршрут через loopback не найден в таблице tproxy_table${NC}"
    fi
    
    # 4. Проверяем, остались ли ещё правила в таблице tproxy_table
    if ip route show table tproxy_table | grep -q .; then
        echo -e "   ${YELLOW}⚠️ В таблице tproxy_table остались маршруты:${NC}"
        ip route show table tproxy_table | sed 's/^/      /'
    else
        echo -e "   ${GREEN}✅ Таблица tproxy_table пуста${NC}"
    fi
    
    return 0
}

# Проверка и создание таблицы маршрутизации если её нет
ensure_tproxy_table() {
    if ! grep -q "tproxy_table" /etc/iproute2/rt_tables; then
        echo "100   tproxy_table" >> /etc/iproute2/rt_tables
        echo -e "${GREEN}✅ Таблица tproxy_table создана${NC}"
    fi
}

# ======================================================
# ФУНКЦИИ УПРАВЛЕНИЯ TINYPROXY
# ======================================================

# Настройка Tinyproxy на использование прокси Минцифры
enable_tinyproxy_upstream() {
    echo -e "${BLUE}🖧 Настройка Tinyproxy...${NC}"
    
    # Обновляем upstream в конфигурации
    sed -i "s/option via.*/option via \"$PROXY_SERVER:$PROXY_PORT\"/" $TINYPROXY_CONFIG
    
    # Перезапускаем Tinyproxy
    /etc/init.d/tinyproxy restart >/dev/null 2>&1
    
    if [ $? -eq 0 ]; then
        echo -e "   ${GREEN}✅ Tinyproxy перезапущен, upstream: $PROXY_SERVER:$PROXY_PORT${NC}"
    else
        echo -e "   ${RED}❌ Ошибка перезапуска Tinyproxy${NC}"
        return 1
    fi
    
    return 0
}

# Отключение upstream в Tinyproxy (прямой доступ)
disable_tinyproxy_upstream() {
    echo -e "${BLUE}🖧 Отключение upstream в Tinyproxy...${NC}"
    
    # Убираем upstream из конфигурации
    sed -i 's/option via.*/option via ""/' $TINYPROXY_CONFIG
    
    # Перезапускаем Tinyproxy
    /etc/init.d/tinyproxy restart >/dev/null 2>&1
    
    if [ $? -eq 0 ]; then
        echo -e "   ${GREEN}✅ Tinyproxy перезапущен, upstream отключен${NC}"
    else
        echo -e "   ${RED}❌ Ошибка перезапуска Tinyproxy${NC}"
        return 1
    fi
    
    return 0
}

# ======================================================
# ФУНКЦИИ ДЛЯ КОМАНД УПРАВЛЕНИЯ
# ======================================================

# Команда: включить прокси
cmd_on() {
    echo -e "${GREEN}========================================${NC}"
    echo -e "${GREEN}  ВКЛЮЧЕНИЕ ПРОКСИ МИНЦИФРЫ (Карелия)  ${NC}"
    echo -e "${GREEN}========================================${NC}"
    
    # Проверка таблицы маршрутизации
    # ensure_tproxy_table

    # Проверка требований
    check_requirements
    if [ $? -ne 0 ]; then
        echo -e "${RED}❌ Ошибка: не все требования выполнены${NC}"
        return 1
    fi
    
    # Проверка доступности прокси
    echo -e "${BLUE}🌐 Проверка доступности прокси-сервера...${NC}"
    if check_proxy_availability; then
        echo -e "   ${GREEN}✅ Прокси-сервер $PROXY_SERVER доступен${NC}"
    else
        echo -e "   ${YELLOW}⚠️ Прокси-сервер $PROXY_SERVER не отвечает${NC}"
        echo -e "   ${YELLOW}   Включение продолжится, но проверьте подключение к сети Минцифры${NC}"
    fi
    
    # Настройка Tinyproxy
    enable_tinyproxy_upstream
    if [ $? -ne 0 ]; then
        echo -e "${RED}❌ Ошибка настройки Tinyproxy${NC}"
        return 1
    fi
    
    # Настройка nftables и маршрутизации
    enable_nft_routing
    if [ $? -ne 0 ]; then
        echo -e "${RED}❌ Ошибка настройки nftables${NC}"
        return 1
    fi
    
    echo -e "${GREEN}========================================${NC}"
    echo -e "${GREEN}✅ ПРОКСИ МИНЦИФРЫ УСПЕШНО ВКЛЮЧЕН${NC}"
    echo -e "${GREEN}========================================${NC}"
    echo "   Сервер: $PROXY_SERVER:$PROXY_PORT"
    echo "   Локальный IP: $LAN_IP"
    echo "   Подсеть: $LAN_NET"
}

# Команда: отключить прокси
cmd_off() {
    echo -e "${YELLOW}========================================${NC}"
    echo -e "${YELLOW}  ОТКЛЮЧЕНИЕ ПРОКСИ МИНЦИФРЫ           ${NC}"
    echo -e "${YELLOW}========================================${NC}"
    
    # Отключение nftables и маршрутизации
    disable_nft_routing
    
    # Отключение upstream в Tinyproxy
    disable_tinyproxy_upstream
    
    echo -e "${GREEN}========================================${NC}"
    echo -e "${GREEN}✅ ПРОКСИ МИНЦИФРЫ ОТКЛЮЧЕН${NC}"
    echo -e "${GREEN}========================================${NC}"
    echo "   Интернет работает напрямую"
}

# Команда: показать статус
cmd_status() {
    echo -e "${BLUE}========================================${NC}"
    echo -e "${BLUE}        СТАТУС ПРОКСИ МИНЦИФРЫ         ${NC}"
    echo -e "${BLUE}========================================${NC}"
    
    # Информация о сертификате
    echo -e "\n${GREEN}📜 Сертификат Ростелекома:${NC}"
    if [ -f /etc/ssl/certs/school/ca-root.crt ]; then
        CERT_INFO=$(openssl x509 -in /etc/ssl/certs/school/ca-root.crt -text -noout 2>/dev/null | grep "Subject:" | head -1)
        echo "   $CERT_INFO"
        echo -e "   ${GREEN}✅ Сертификат установлен${NC}"
    else
        echo -e "   ${RED}❌ Сертификат не найден${NC}"
    fi
    
    # Статус nftables
    echo -e "\n${GREEN}🔒 nftables:${NC}"
    if nft list table $NFT_TABLE_NAME >/dev/null 2>&1; then
        RULES_COUNT=$(nft list table $NFT_TABLE_NAME 2>/dev/null | grep -c "chain\|set")
        echo -e "   ${GREEN}✅ Правила загружены (правил: $RULES_COUNT)${NC}"
    else
        echo -e "   ${RED}❌ Правила не загружены${NC}"
    fi
    
    # Статус Tinyproxy
    echo -e "\n${GREEN}🖧 Tinyproxy:${NC}"
    if pgrep tinyproxy >/dev/null; then
        echo -e "   ${GREEN}✅ Запущен${NC}"
        UPSTREAM=$(grep "option via" $TINYPROXY_CONFIG | grep -v '""' | cut -d'"' -f2)
        if [ -n "$UPSTREAM" ]; then
            echo "   Upstream: $UPSTREAM"
            if [ "$UPSTREAM" = "$PROXY_SERVER:$PROXY_PORT" ]; then
                echo -e "   ${GREEN}✅ Настроен на прокси Минцифры${NC}"
            else
                echo -e "   ${YELLOW}⚠️ Настроен на другой прокси: $UPSTREAM${NC}"
            fi
        else
            echo -e "   ${YELLOW}⚠️ Upstream не настроен (прямой доступ)${NC}"
        fi
    else
        echo -e "   ${RED}❌ Не запущен${NC}"
    fi
    
    # Статус маршрутизации
    echo -e "\n${GREEN}🚦 Маршрутизация TPROXY:${NC}"
    if ip rule show | grep -q "fwmark 0x1"; then
        RULE_INFO=$(ip rule show | grep "fwmark 0x1" | head -1)
        echo -e "   ${GREEN}✅ Правило fwmark настроено${NC}"
        echo "   $RULE_INFO"
    else
        echo -e "   ${RED}❌ Правило fwmark не настроено${NC}"
    fi
    
    if ip route show table tproxy_table | grep -q "local default dev lo"; then
        echo -e "   ${GREEN}✅ Маршрут через loopback настроен${NC}"
    else
        echo -e "   ${RED}❌ Маршрут через loopback не настроен${NC}"
    fi
    
    # Доступность прокси
    echo -e "\n${GREEN}🌐 Доступность прокси Минцифры:${NC}"
    if check_proxy_availability; then
        echo -e "   ${GREEN}✅ Прокси-сервер $PROXY_SERVER доступен${NC}"
    else
        echo -e "   ${RED}❌ Прокси-сервер $PROXY_SERVER НЕ доступен${NC}"
        echo "   Проверьте подключение к сети Минцифры"
    fi
    
    # Проверка работы через тестовый запрос
    echo -e "\n${GREEN}🧪 Быстрый тест:${NC}"
    TEST_RESULT=$(curl -s -o /dev/null -w "%{http_code}" -I --max-time 3 http://google.com 2>/dev/null)
    if [ "$TEST_RESULT" = "200" ] || [ "$TEST_RESULT" = "301" ] || [ "$TEST_RESULT" = "302" ]; then
        echo -e "   ${GREEN}✅ Интернет работает (HTTP $TEST_RESULT)${NC}"
    else
        echo -e "   ${RED}❌ Интернет не работает (код $TEST_RESULT)${NC}"
    fi
}

# Команда: показать логи
cmd_logs() {
    echo -e "${BLUE}📋 ЛОГИ TINYPROXY (последние 50 строк)${NC}"
    echo "========================================"
    if [ -f /var/log/tinyproxy.log ]; then
        tail -50 /var/log/tinyproxy.log
    else
        echo "Лог-файл не найден"
    fi
    
    echo -e "\n${BLUE}📋 ЛОГИ NFTABLES (последние 20 строк)${NC}"
    echo "========================================"
    logread | grep -i nft | tail -20
}

# Команда: тестирование
cmd_test() {
    echo -e "${BLUE}========================================${NC}"
    echo -e "${BLUE}        ТЕСТИРОВАНИЕ ПРОКСИ            ${NC}"
    echo -e "${BLUE}========================================${NC}"
    
    # Проверка HTTP
    echo -e "\n${GREEN}1. Проверка HTTP (порт 80):${NC}"
    HTTP_CODE=$(curl -x $LAN_IP:8888 -s -o /dev/null -w "%{http_code}" --max-time 5 http://httpbin.org/ip 2>/dev/null)
    if [ "$HTTP_CODE" = "200" ]; then
        HTTP_IP=$(curl -x $LAN_IP:8888 -s --max-time 5 http://httpbin.org/ip 2>/dev/null | grep -o '"origin": "[^"]*"' | cut -d'"' -f4)
        echo -e "   ${GREEN}✅ Успешно (код $HTTP_CODE)${NC}"
        echo "   Ваш IP через прокси: $HTTP_IP"
    else
        echo -e "   ${RED}❌ Ошибка (код $HTTP_CODE)${NC}"
    fi
    
    # Проверка HTTPS
    echo -e "\n${GREEN}2. Проверка HTTPS (порт 443):${NC}"
    HTTPS_CODE=$(curl -x $LAN_IP:8888 -s -o /dev/null -w "%{http_code}" --max-time 5 https://httpbin.org/ip 2>/dev/null)
    if [ "$HTTPS_CODE" = "200" ]; then
        HTTPS_IP=$(curl -x $LAN_IP:8888 -s --max-time 5 https://httpbin.org/ip 2>/dev/null | grep -o '"origin": "[^"]*"' | cut -d'"' -f4)
        echo -e "   ${GREEN}✅ Успешно (код $HTTPS_CODE)${NC}"
        echo "   Ваш IP через прокси: $HTTPS_IP"
    else
        echo -e "   ${RED}❌ Ошибка (код $HTTPS_CODE)${NC}"
        echo "   Возможные причины:"
        echo "   - Проблемы с сертификатом"
        echo "   - Прокси не работает с HTTPS"
        echo "   - Таймаут соединения"
    fi
    
    # Проверка через curl напрямую (без прокси) для сравнения
    echo -e "\n${GREEN}3. Проверка без прокси (прямой доступ):${NC}"
    DIRECT_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 https://httpbin.org/ip 2>/dev/null)
    if [ "$DIRECT_CODE" = "200" ]; then
        DIRECT_IP=$(curl -s --max-time 5 https://httpbin.org/ip 2>/dev/null | grep -o '"origin": "[^"]*"' | cut -d'"' -f4)
        echo -e "   ${GREEN}✅ Успешно (код $DIRECT_CODE)${NC}"
        echo "   Ваш прямой IP: $DIRECT_IP"
    else
        echo -e "   ${RED}❌ Ошибка (код $DIRECT_CODE)${NC}"
    fi
}

# Команда: показать правила
cmd_show() {
    echo -e "${BLUE}📋 ТЕКУЩИЕ ПРАВИЛА NFTABLES${NC}"
    echo "========================================"
    if nft list table $NFT_TABLE_NAME >/dev/null 2>&1; then
        nft list table $NFT_TABLE_NAME
    else
        echo "Правила не загружены"
    fi
    
    echo -e "\n${BLUE}📋 ПРАВИЛА МАРШРУТИЗАЦИИ${NC}"
    echo "========================================"
    echo "Правила fwmark:"
    ip rule show | grep "fwmark" || echo "   Нет правил с fwmark"
    echo ""
    echo "Таблица tproxy_table:"
    ip route show table tproxy_table 2>/dev/null || echo "   Таблица tproxy_table пуста"
}

# Команда: помощь
cmd_help() {
    echo -e "${BLUE}========================================${NC}"
    echo -e "${BLUE}  УПРАВЛЕНИЕ ПРОКСИ МИНЦИФРЫ           ${NC}"
    echo -e "${BLUE}========================================${NC}"
    echo ""
    echo "Использование: proxy {on|off|status|logs|test|show|restart|help}"
    echo ""
    echo "Команды:"
    echo "  on          - включить прокси Минцифры (10.0.10.52:3128)"
    echo "  off         - отключить прокси (прямой доступ)"
    echo "  status      - показать статус всех компонентов"
    echo "  logs        - показать логи Tinyproxy и nftables"
    echo "  test        - протестировать работу прокси"
    echo "  show        - показать текущие правила nftables"
    echo "  restart     - перезапустить прокси (off + on)"
    echo "  help        - показать эту справку"
    echo ""
    echo "Примеры:"
    echo "  proxy on    # включить прокси"
    echo "  proxy off   # отключить прокси"
    echo "  proxy test  # проверить работу"
    echo ""
    echo "Текущие настройки:"
    echo "  Прокси сервер: $PROXY_SERVER:$PROXY_PORT"
    echo "  Локальный IP: $LAN_IP"
    echo "  Подсеть: $LAN_NET"
}

# Команда: перезапуск
cmd_restart() {
    echo -e "${YELLOW}🔄 ПЕРЕЗАПУСК ПРОКСИ МИНЦИФРЫ${NC}"
    cmd_off
    sleep 2
    cmd_on
}

# ======================================================
# ГЛАВНАЯ ФУНКЦИЯ - ОБРАБОТКА КОМАНД
# ======================================================

case "$1" in
    on|enable|start)
        cmd_on
        ;;
    off|disable|stop)
        cmd_off
        ;;
    status)
        cmd_status
        ;;
    logs)
        cmd_logs
        ;;
    test)
        cmd_test
        ;;
    show|rules)
        cmd_show
        ;;
    restart)
        cmd_restart
        ;;
    help|--help|-h)
        cmd_help
        ;;
    *)
        echo -e "${RED}❌ Неизвестная команда: $1${NC}"
        cmd_help
        exit 1
        ;;
esac

exit 0
EOF

# Делаем скрипт исполняемым
chmod +x /root/mintsifra-proxy.sh

# Создаем символическую ссылку для удобства
ln -sf /root/mintsifra-proxy.sh /usr/bin/proxy

echo -e "${GREEN}✅ Скрипт управления установлен!${NC}"
echo -e "Теперь вы можете использовать команду: ${YELLOW}proxy${NC}"
echo -e "Пример: ${YELLOW}proxy help${NC} - для справки"
EOF

# Делаем скрипт исполняемым
chmod +x /root/mintsifra-proxy.sh

# Создаем символическую ссылку для удобства
ln -sf /root/mintsifra-proxy.sh /usr/bin/proxy
```

8. Запуск и проверка

8.1. Проверьте ваш IP и подсеть
```bash
# Убедитесь, что в скрипте правильные IP и подсеть
# При необходимости отредактируйте:
vi /root/mintsifra-proxy.sh
# Исправьте переменные LAN_IP и LAN_NET в начале файла
```

8.2. Запустите прокси
```bash
# Запуск прокси
/root/mintsifra-proxy.sh start

# Проверка статуса
/root/mintsifra-proxy.sh status
```

8.3. Проверьте логи
```bash
/root/mintsifra-proxy.sh logs
```

8.4. Протестируйте работу
```bash
/root/mintsifra-proxy.sh test
```

8.5. Проверка с клиентского устройства
На любом компьютере или ноутбуке в вашей сети (БЕЗ настроек прокси) выполните:
```bash
# Проверка внешнего IP (должен показать IP Минцифры)
curl http://httpbin.org/ip
curl https://httpbin.org/ip
```

8.6. Проверка через браузер
Откройте любой сайт, например https://2ip.ru - должен показывать IP Минцифры, а не ваш реальный IP.


🔄 Теперь как это будет работать:
При выполнении proxy on:

    ✅ Проверит, существует ли таблица tproxy_table, создаст если нет

    ✅ Проверит, загружены ли правила nftables

    ✅ Если правила не загружены - загрузит их из /root/tinyproxy-tproxy.nft

    ✅ Если правила уже загружены - пропустит загрузку

    ✅ Настроит правила маршрутизации (если не настроены)

При выполнении proxy off:

    ❌ Удалит правила nftables (если они загружены)

    ❌ Удалит правила маршрутизации (если они есть)


```bash
# Сначала отключим прокси (если был включен)
proxy off

# Проверим, что правила удалены
nft list table inet tinyproxy  # Должно быть ошибкой или пусто

# Включим прокси
proxy on

# Проверим, что правила загрузились
nft list table inet tinyproxy  # Должны увидеть таблицу

# Проверим статус
proxy status

# Протестируем
proxy test
```

9. Диагностика проблем

9.1. Прокси-сервер 10.0.10.52 не доступен
```bash
# Проверка доступности
ping 10.0.10.52

# Если не пингуется, проверьте маршрут
traceroute -n 10.0.10.52
```
Решение: Прокси Минцифры доступен только из сети Минцифры. Если вы не в ней, нужно:
 - Подключиться через VPN до сети Минцифры
 - Или использовать роутер только в учреждениях, где есть сеть Минцифры

9.2. Tinyproxy не запускается
```bash
# Проверка логов
logread | grep tinyproxy

# Проверка конфигурации
cat /etc/config/tinyproxy
```

9.3. nftables правила не загружаются
```bash
# Проверка синтаксиса
nft -c -f /root/tinyproxy-tproxy.nft

# Проверка загруженных правил
nft list ruleset
```

9.4. DNS не работает
```bash
# Проверка статуса dnsmasq
/etc/init.d/dnsmasq status

# Проверка логов
logread | grep dnsmasq
```

9.5. Полный сброс и перезапуск
Если всё сломалось, выполните:
```bash
# Остановить прокси
/root/mintsifra-proxy.sh stop

# Перезагрузить firewall
/etc/init.d/firewall restart

# Перезагрузить Tinyproxy
/etc/init.d/tinyproxy restart

# Запустить прокси заново
/root/mintsifra-proxy.sh start
```
