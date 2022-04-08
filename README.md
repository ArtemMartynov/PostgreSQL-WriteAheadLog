# Подготовка ОС
1. Создать две виртуальные машины, с двумя сетевыми интерфейсами.
Один служебный для взаимодействия машин между собой (внутренний), другой для доступа в сеть Интернет (внешний).
2. Внешний интерфейс будет использовать виртуальный сетевой адрес (На период конфигурации серверов, машины имеют два различных адреса).
3. Начальная сетевая конфигурация имеет следующий вид:
PGSQL-1 
ens33 (внешний)
ip-адрес: 192.168.1.124/24
Gateway: 192.168.1.1
ens35 (служебный)
ip-адрес: 192.168.2.11/24

PGSQL-2
ens33 (внешний)
ip-адрес: 192.168.1.65/24
Gateway: 192.168.1.1
ens35 (служебный)
ip-адрес: 192.168.2.12/24

# Установка пакетов
1. apt install net-tools
2. apt install postresql

# Настройка PostgreSQL(primary)
1.Создание пользователя для репликации БД: su - postgres -с "createuser -U postgres repuser -P -c 5 --replication"
2.
host replication repuser 192.168.2.11/32 trust
host replication repuser 192.168.2.12/32 trust
host all postgres 192.168.1.113/32 trust
3.
listen_addresses = '*'

# Настройка PostgreSQL(standby)
1.
host replication repuser 192.168.2.11/32 trust
host replication repuser 192.168.2.12/32 trust
host all postgres 192.168.1.113/32 trust
2.
mv /var/lib/postgresql/11/main/ /var/lib/postgresql/11/main_old/
3.
systemctl stop postgresql
4.
su - postgres -c "pg_basebackup -h 192.168.2.11 -D /var/lib/postgresql/11/main/ -U repuser -w --wal-method=stream
5.
systemctl start postgresql

# Написание скриптов
1.
#!/bin/bash

#ps - переменная статуса postgresql на дополнительном сервере БД (1 - вкл, 0 - выкл)
#fl - переменная статуса порта 5432 на основном сервере БД (10 - открыт и доступен, 3 - недоступен)
#Если fl = 3 и ps = 1, то основной сервер недоступен, а postgresql на дополнительном работает -> выход из скрипта
#Если fl = 10 и ps = 0, то основной сервер доступен, а postgresql на дополнительном не работает -> репликация каталога /data с основного на дополнительный
#Если fl = 3 и ps = 0, то основной сервер недоступен, а postgresql на дополнительном не работает -> включение второго интерфейса и Postgresql на дополнительном
#Если fl = 10 и ps = 1, то основной сервер доступен, а postgresql на дополнительном работает -> репликация с дополнительного на основной каталога /data, отключение postgresql и второго интерфейса на дополнительном

sleep 10

#Декларирование переменных
declare -i ps
declare -i fl
DATA_FOLDER="/var/lib/postgresql/11"

#Проверяем состояние службы postgresql на резервном сервере, если работает ps=1, иначе 0
systemctl is-active postgresql>/dev/null 2>&1 && let "ps = 1" || let "ps = 0"

iface='192.168.2.11' # IP адрес основного сервера

#Проверяем доступность порта 5432 на основном сервере, если доступен fl=10, иначе 1
(echo > /dev/tcp/$iface/5432) >& /dev/null && let "fl = 10" || let "fl = 0"

#Цикл проверки доступности порта 5432 на основном сервере каждые 10 секунд проверяем доступность
while [[ $fl -lt 3 ]]
do
	sleep 10; (echo > /dev/tcp/$iface/5432) >& /dev/null && let "fl = 10" || let "fl = $fl + 1"	
done

#Получаем сумму двух параметров, на которые ориентируемся
let "sum = $fl + $ps"

#Case по суммам параметров (см. параметры выше)
case $sum in
	3)
	systemctl start postgresql; ifconfig ens33 up; 
	echo "$(date) Primary dead, starting reserve... (event 3)" >> /var/log/HA_slave.log; exit 0;;

	4)
	echo "$(date) Primary dead, reserve working, waiting... (event 4)" >> /var/log/HA_slave.log; exit 0;;

	10)
	rm -rf $DATA_FOLDER/main_old && mv $DATA_FOLDER/main $DATA_FOLDER/main_old
	su - postgres -c "pg_basebackup -h $iface -D $DATA_FOLDER/main/ -U repuser -w --wal-method=stream >& /dev/null"; 
	echo "$(date) Primary alive, replicating of /data from primary to reserve... (event 10)">> /var/log/HA_slave.log; exit 0;;

	11)
	systemctl stop postgresql; ifconfig ens33 down; 
	echo "$(date) Primary alive, shutting down reserve... (event 11)">> /var/log/HA_slave.log; exit 0;;
esac

2.
#!/bin/bash

#Значения параметров
#fl = 10 - сеть доступна
#fl = 3 - сеть недоступна
#ps = 1 - служба работает
#ps = 0 - служба не работает

declare -i ps
declare -i fl
DATA_FOLDER="/var/lib/postgresql/11"
#Проверяем состояние службы postgresql на основном сервере, если работает ps=1, иначе 0
systemctl is-active postgresql>/dev/null 2>&1 && let "ps = 1" || let "ps = 0"

IP=("8.8.8.8") # IP какой-нибудь машины
fl=0
pattern="0 received"
while [[ $fl -lt 3 ]]
do
	result=$(ping -c 2 -W 1 -q $IP | grep transmitted)
	if [[ $result =~ $pattern || $? -ne 0 ]]; then
		sleep 10;let "fl = $fl + 1"	 
	else
		let "fl = 10"
	fi
done

#Получаем сумму двух параметров, на которые ориентируемся
let "sum = $fl + $ps"

#Case по суммам параметров (см. параметры выше)
case $sum in

	# Служба на основном сервере не работает, сеть недоступна
	3)
	echo "$(date) Primary dead, waiting...( event 3 )" >> /var/log/HA_master.log;exit 0;;
	# Служба на основном сервере работает, сеть доступна
	4)
	echo "$(date) Primary dead, stopping postgresql... (event 4)" >> /var/log/HA_master.log;systemctl stop postgresql; exit 0;;
	# Служба не работает, сеть доступна
	10)
	rm -rf $DATA_FOLDER/main_old && mv $DATA_FOLDER/main $DATA_FOLDER/main_old
	su - postgres -c "pg_basebackup -h 192.168.2.12 -D $DATA_FOLDER/main/ -U repuser -w --wal-method=stream >& /dev/null"; systemctl start postgresql; 
	echo "$(date) Primary alive, starting postgresql... (event 10)" >> /var/log/HA_master.log;exit 0;;
	# Сулжба работает, сеть доступна
	11)
	echo "$(date) Primary alive, waiting... (event 11)" >> /var/log/HA_master.log;exit 0;;
esac

# Настройка расписания запуска скриптов в crontab
1. cp /root/HA_primary.sh /usr/local/bin/HA_primary
2. cp /root/HA_standby.sh /usr/local/bin/HA_standby
3. /etc/crontab: * * * * * root /usr/local/bin/HA_primary
4. /etc/crontab: * * * * * root /usr/local/bin/HA_standby

# Проверка

1.
import time
from _datetime import datetime
import psycopg2


def main():
    sql = "SELECT * FROM guestbook;"

    i = 0
    while True:
        try:
            client = psycopg2.connect(
                dbname='postgres',
                user='postgres',
                password='toor',
                host='192.168.1.124',
                port=5432,
                connect_timeout=5
            )
            cursor = client.cursor()
            cursor.execute(sql)
            data: list = cursor.fetchall()
            cursor.close()

            print("%02d" % i, datetime.now(), data)
        except Exception as _e:
            print("%02d" % i, datetime.now(), "База данных недоступна")

        i += 1
        time.sleep(1)


if __name__ == '__main__':
    main()
