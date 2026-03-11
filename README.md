# Настройка OpenWRT для прокси Минцифры

Ссылка на сертификат ```url https://espd.rt.ru/settings/cert-install```

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


6. Настройка маршрутизации для TPROXY
6.1. Создание таблицы маршрутизации
```bash
# Добавляем таблицу tproxy_table, если её нет
if ! grep -q "tproxy_table" /etc/iproute2/rt_tables; then
    echo "100   tproxy_table" >> /etc/iproute2/rt_tables
    echo "✅ Таблица tproxy_table добавлена"
else
    echo "✅ Таблица tproxy_table уже существует"
fi
```

6.2. Добавление правил маршрутизации
```bash
# Добавляем правило для пакетов с меткой fwmark 1
ip -4 rule add fwmark 1 lookup tproxy_table priority 1000 2>/dev/null

# Добавляем маршрут через loopback
ip -4 route add local default dev lo table tproxy_table 2>/dev/null

# Проверяем
echo "✅ Правила маршрутизации:"
ip rule show | grep fwmark
ip route show table tproxy_table
```

6.3. Делаем правила постоянными
```bash
# Добавляем в rc.local для автоматического применения при загрузке
cat >> /etc/rc.local << 'EOF'

# TPROXY routing setup
ip -4 rule add fwmark 1 lookup tproxy_table priority 1000 2>/dev/null
ip -4 route add local default dev lo table tproxy_table 2>/dev/null
EOF

# Проверяем
tail -5 /etc/rc.local
```

7. Настройка nftables для прозрачного прокси
7.1. Создание правил nftables
```bash
# Создаем директорию для правил
mkdir -p /etc/nftables.d

# Создаем файл с правилами
cat > /etc/nftables.d/11-tinyproxy-tproxy.nft << 'EOF'
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
chmod +x /etc/nftables.d/11-tinyproxy-tproxy.nft
```

7.2. Проверка синтаксиса и загрузка правил
```bash
# Проверяем синтаксис
nft -c -f /etc/nftables.d/11-tinyproxy-tproxy.nft

# Если синтаксис правильный, применяем правила
/etc/init.d/firewall restart

# Проверяем, что таблица создалась
nft list table inet tinyproxy
```

8. Создание скрипта управления
8.1. Создаем удобный скрипт для управления прокси
```bash
cat > /root/mintsifra-proxy.sh << 'EOF'
#!/bin/sh

# Прокси Минцифры для Республики Карелия (код 10)
PROXY_SERVER="10.0.10.52"
PROXY_PORT="3128"
LAN_IP="192.168.1.1"  # ЗАМЕНИТЕ НА ВАШ IP
LAN_NET="192.168.1.0/24"  # ЗАМЕНИТЕ НА ВАШУ ПОДСЕТЬ

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

case "$1" in
    start)
        echo -e "${GREEN}▶ Включение прокси Минцифры (Карелия)...${NC}"
        
        # Обновляем upstream в tinyproxy
        sed -i "s/option via.*/option via \"$PROXY_SERVER:$PROXY_PORT\"/" /etc/config/tinyproxy
        
        /etc/init.d/tinyproxy restart
        
        # Загружаем правила nftables
        nft -f /etc/nftables.d/11-tinyproxy-tproxy.nft 2>/dev/null
        
        echo -e "${GREEN}✅ Прокси Минцифры включен${NC}"
        echo "   Сервер: $PROXY_SERVER:$PROXY_PORT"
        ;;
        
    stop)
        echo -e "${YELLOW}▶ Отключение прокси Минцифры...${NC}"
        
        # Удаляем правила nftables
        nft delete table inet tinyproxy 2>/dev/null
        
        # Отключаем прокси в Tinyproxy
        sed -i 's/option via.*/option via ""/' /etc/config/tinyproxy
        /etc/init.d/tinyproxy restart
        
        echo -e "${GREEN}✅ Прокси отключен, интернет работает напрямую${NC}"
        ;;
        
    status)
        echo -e "${GREEN}=================================${NC}"
        echo -e "${GREEN}   СТАТУС ПРОКСИ МИНЦИФРЫ       ${NC}"
        echo -e "${GREEN}=================================${NC}"
        
        # Проверка nftables
        if nft list table inet tinyproxy >/dev/null 2>&1; then
            echo -e "${GREEN}✅ nftables правила: ЗАГРУЖЕНЫ${NC}"
        else
            echo -e "${RED}❌ nftables правила: НЕ ЗАГРУЖЕНЫ${NC}"
        fi
        
        # Проверка Tinyproxy
        if pgrep tinyproxy >/dev/null; then
            echo -e "${GREEN}✅ Tinyproxy: ЗАПУЩЕН${NC}"
            UPSTREAM=$(grep "option via" /etc/config/tinyproxy | grep -v '""' | cut -d'"' -f2)
            if [ -n "$UPSTREAM" ]; then
                echo "   Прокси Минцифры: $UPSTREAM"
                if [ "$UPSTREAM" = "$PROXY_SERVER:$PROXY_PORT" ]; then
                    echo -e "   ${GREEN}✅ Настроен на Карелию (код 10)${NC}"
                else
                    echo -e "   ${YELLOW}⚠️ Настроен на другой прокси: $UPSTREAM${NC}"
                fi
            else
                echo -e "   ${RED}❌ Прокси Минцифры: не настроен${NC}"
            fi
        else
            echo -e "${RED}❌ Tinyproxy: НЕ ЗАПУЩЕН${NC}"
        fi
        
        # Проверка маршрутизации
        if ip rule show | grep -q "fwmark 0x1"; then
            echo -e "${GREEN}✅ Маршрутизация: НАСТРОЕНА${NC}"
        else
            echo -e "${RED}❌ Маршрутизация: НЕ НАСТРОЕНА${NC}"
        fi
        
        # Проверка доступности прокси Минцифры
        echo -e "${GREEN}---------------------------------${NC}"
        echo "Проверка доступности прокси-сервера:"
        if ping -c 2 -W 2 $PROXY_SERVER >/dev/null 2>&1; then
            echo -e "${GREEN}✅ Прокси-сервер $PROXY_SERVER ДОСТУПЕН${NC}"
        else
            echo -e "${RED}❌ Прокси-сервер $PROXY_SERVER НЕ ДОСТУПЕН${NC}"
            echo "   Возможно, требуется VPN или маршрут до сети Минцифры"
        fi
        ;;
        
    logs)
        echo -e "${GREEN}📋 Последние 30 строк лога Tinyproxy:${NC}"
        echo "---------------------------------"
        tail -30 /var/log/tinyproxy.log 2>/dev/null || echo "Лог-файл не найден"
        echo ""
        echo -e "${GREEN}📋 Логи nftables:${NC}"
        logread | grep -i nft | tail -10
        ;;
        
    test)
        echo -e "${GREEN}🧪 ТЕСТИРОВАНИЕ ПРОКСИ${NC}"
        echo "---------------------------------"
        
        # Проверка HTTP
        echo "1. Проверка HTTP через прокси:"
        HTTP_RESULT=$(curl -x $LAN_IP:8888 -s -o /dev/null -w "%{http_code}" http://httpbin.org/ip 2>/dev/null)
        if [ "$HTTP_RESULT" = "200" ]; then
            echo -e "   ${GREEN}✅ HTTP: OK (код $HTTP_RESULT)${NC}"
        else
            echo -e "   ${RED}❌ HTTP: ОШИБКА (код $HTTP_RESULT)${NC}"
        fi
        
        # Проверка HTTPS
        echo "2. Проверка HTTPS через прокси:"
        HTTPS_RESULT=$(curl -x $LAN_IP:8888 -s -o /dev/null -w "%{http_code}" https://httpbin.org/ip 2>/dev/null)
        if [ "$HTTPS_RESULT" = "200" ]; then
            echo -e "   ${GREEN}✅ HTTPS: OK (код $HTTPS_RESULT)${NC}"
        else
            echo -e "   ${RED}❌ HTTPS: ОШИБКА (код $HTTPS_RESULT)${NC}"
        fi
        
        # Внешний IP
        echo "3. Ваш внешний IP через прокси:"
        EXTERNAL_IP=$(curl -x $LAN_IP:8888 -s http://httpbin.org/ip 2>/dev/null | grep -o '"origin": "[^"]*"' | cut -d'"' -f4)
        if [ -n "$EXTERNAL_IP" ]; then
            echo -e "   ${GREEN}✅ IP: $EXTERNAL_IP${NC}"
        else
            echo -e "   ${RED}❌ Не удалось определить IP${NC}"
        fi
        ;;
        
    help|*)
        echo "Использование: $0 {start|stop|status|logs|test|help}"
        echo ""
        echo "Параметры:"
        echo "  start   - включить прокси Минцифры (10.0.10.52:3128)"
        echo "  stop    - отключить прокси"
        echo "  status  - показать статус всех компонентов"
        echo "  logs    - показать логи"
        echo "  test    - протестировать работу прокси"
        echo "  help    - показать эту справку"
        ;;
esac
EOF

# Делаем скрипт исполняемым
chmod +x /root/mintsifra-proxy.sh

# Создаем символическую ссылку для удобства
ln -sf /root/mintsifra-proxy.sh /usr/bin/proxy
```

9. Запуск и проверка
9.1. Проверьте ваш IP и подсеть
```bash
# Убедитесь, что в скрипте правильные IP и подсеть
# При необходимости отредактируйте:
vi /root/mintsifra-proxy.sh
# Исправьте переменные LAN_IP и LAN_NET в начале файла
```

9.2. Запустите прокси
```bash
# Запуск прокси
/root/mintsifra-proxy.sh start

# Проверка статуса
/root/mintsifra-proxy.sh status
```

9.3. Проверьте логи
```bash
/root/mintsifra-proxy.sh logs
```

9.4. Протестируйте работу
```bash
/root/mintsifra-proxy.sh test
```

9.5. Проверка с клиентского устройства
На любом компьютере или ноутбуке в вашей сети (БЕЗ настроек прокси) выполните:
```bash
# Проверка внешнего IP (должен показать IP Минцифры)
curl http://httpbin.org/ip
curl https://httpbin.org/ip
```

9.6. Проверка через браузер
Откройте любой сайт, например https://2ip.ru - должен показывать IP Минцифры, а не ваш реальный IP.

10. Диагностика проблем
10.1. Прокси-сервер 10.0.10.52 не доступен
```bash
# Проверка доступности
ping 10.0.10.52

# Если не пингуется, проверьте маршрут
traceroute -n 10.0.10.52
```
Решение: Прокси Минцифры доступен только из сети Минцифры. Если вы не в ней, нужно:
 - Подключиться через VPN до сети Минцифры
 - Или использовать роутер только в учреждениях, где есть сеть Минцифры

10.2. Tinyproxy не запускается
```bash
# Проверка логов
logread | grep tinyproxy

# Проверка конфигурации
cat /etc/config/tinyproxy
```

10.3. nftables правила не загружаются
```bash
# Проверка синтаксиса
nft -c -f /etc/nftables.d/11-tinyproxy-tproxy.nft

# Проверка загруженных правил
nft list ruleset
```

10.4. DNS не работает
```bash
# Проверка статуса dnsmasq
/etc/init.d/dnsmasq status

# Проверка логов
logread | grep dnsmasq
```

10.5. Полный сброс и перезапуск
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
