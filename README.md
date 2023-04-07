# keenetic-ipset

Что это и для чего? Для обхода блокировок российских подсетей посредством пропуска их через ВПН соединение

Требования:
```
Установленный OPKG
Запущенное подключение к ВПН на роутере(в инструкции используется W.A.R.P через протокол WireGuard)
```


Подготовка:
Подключаемся к SSH OPKG(не путать с SSH Кинетика),по умолчанию это 
```
192.168.1.1:222
```
логин
```
root
```
пароль
```
keenetic
```
По классике
```
opkg update
opkg upgrade
```

Устанавливаем необходимые пакеты
```
opkg install ipset 
opkg install iptables
opkg install nano
```


Настраиваем:
```
ipset create rusubnets hash:net
```
Затем создаем скрипт который будет управлять процессом
```
nano /bin/ipset.sh
```
и вставляем в него следующий текст

```
#!/bin/sh

device=nwg0 #указываем имя интерфейса ВПН
file=/opt/etc/ru.cidr #местонахождение файла откуда будут браться заблокированные подсети
ipset=vpn #название списка для ipset
fwmark=1 #цифра маркера которой помечаем трафик
table=222 #номер таблицы роутинга
guest=192.168.2.0/24 #указываем гостевую сеть если используем



ipset create $ipset hash:net -exist

while read line || [ -n "$line" ]; do
  [ -z "$line" ] && continue
  [ "${line:0:1}" = "#" ] && continue

  ipset -exist add $ipset $line
done < $file

ip rule add fwmark $fwmark table $table
ip route add default dev $device table $table
iptables -w -t mangle -A PREROUTING ! -s $guest -m conntrack --ctstate NEW -m set --match-set $ipset dst -j CONNMARK --set-mark $fwmark
iptables -w -t mangle -A PREROUTING ! -s $guest -m set --match-set $ipset dst -j CONNMARK --restore-mark

echo Done
```
Сохраняем,закрываем и даем права на запуск

```
chmod +x opt/bin/ipset.sh
```

Далее создаем задание для демона init.d который будет запускать наш скрипт после перезагрузки роутера

```
nano /opt/etc/init.d/S99Unblock
```
и вставляем в него строки

```
#!/bin/sh

[ "$1" != "start" ] && exit 0

/opt/bin/ipset.sh &
```

Сохраняем и даем права на запуск

```
chmod +x opt/bin/ipset.sh
```

Переходим к загрузке файла с списком заблокированных подсетей
Список берем из репо [Herrbischoff](https://github.com/herrbischoff/country-ip-blocks)

```
wget -O/opt/ru.cidr https://github.com/herrbischoff/country-ip-blocks/raw/ma
ster/ipv4/ru.cidr --no-check-certificate
```

Процесс настройки закончен.
Даем команду 
```
/opt/bin/ipset.sh
```
и ждем какое-то количество времени.На Keenetic Viva 1910 процесс занимает около минуты.Если процесс успешно завершен - увидим Done.
Если же скрипт завершится с ошибкой - перепроверьте не была ли допущена где-то ошибка у вас.

