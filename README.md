# keenetic-ipset

Что это и для чего? Для обхода блокировок российских подсетей посредством пропуска их через ВПН соединение
Важно - если вы использовали Zaborona.help - **удалите** профиль OpenVPN из роутера.Удалите а не отключите.

НЕ ИСПОЛЬЗУЙТЕ С СВОИМ СЕРВЕРОМ ВПН ЕСЛИ ВАШ ХОСТЕР НЕГАТИВНО ОТНОСИТСЯ К ТОРРЕНТ-ТРАФИКУ!Торренты с сидами/пирами из русских подсетей начнут идти через ВПН!И,как следствие,вам сразу же прилетит абуза и вы лишитесь своего сервера.

Также возможно незначительное повышение нагрузки на процессор из-за описанного выше.

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


Создаем скрипт который будет управлять процессом
```
nano /opt/bin/ipset.sh
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

echo Done
```
Сохраняем,закрываем и даем права на запуск

```
chmod +x /opt/bin/ipset.sh
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
chmod +x /opt/etc/init.d/S99Unblock
```

Переходим к загрузке файла с списком заблокированных подсетей
Список берем из репо [Herrbischoff](https://github.com/herrbischoff/country-ip-blocks)

```
wget -O /opt/etc/ru.cidr https://github.com/herrbischoff/country-ip-blocks/raw/master/ipv4/ru.cidr --no-check-certificate  
```
Если wget выдает ошибку - руками возьмите файл отсюда 
```
https://github.com/herrbischoff/country-ip-blocks/raw/master/ipv4/ru.cidr
```

и положите его в 
```
/opt/etc
```


Теперь настроим роутер что бы он не удалял наши правила.Для этого даем в консоль 

```
nano /opt/etc/ndm/netfilter.d/100-fwmark.sh
```
и копируем сюда текст

```
#!/bin/sh

ipset=vpn
fwmark=1
guest=192.168.2.0/24 #указываем диапазон гостевой подсети

[ "$type" == "ip6tables" ] && exit 0
[ "$table" != "mangle" ] && exit 0

if [ -z "$(iptables-save | grep $ipset)" ]; then
  iptables -w -t mangle -A PREROUTING ! -s $guest -m conntrack --ctstate NEW -m set --match-set $ipset dst -j CONNMARK --set-mark $fwmark
  iptables -w -t mangle -A PREROUTING ! -s $guest -m set --match-set $ipset dst -j CONNMARK --restore-mark
fi

exit 0
```
Сохраняем,закрываем.Не забываем про 

```
chmod +x /opt/etc/ndm/netfilter.d/100-fwmark.sh
```

Процесс настройки закончен.
Даем команду 
```
/opt/bin/ipset.sh
```
и ждем какое-то количество времени.На Keenetic Viva 1910 процесс занимает около минуты.Если процесс успешно завершен - увидим Done.
Если же скрипт завершится с ошибкой - перепроверьте не была ли допущена где-то ошибка у вас.
После ребута роутера не нужно запускать задание вручную,все поднимется само.


Возможные ошибки и их методы решения:

**ip: RTNETLINK answers: File exists** - обнаружен дубликат маршрута,удалите подключение к Zaborona и/или другому VPN который маршрутизирует подсети.


**ip: RTNETLINK answers: Network is down** - выбран неправильный/отключенный интерфейс ВПН.Проверьте правильность имени вашего интерфейса.


Автор гайда - [Le Maxime]

Автор метода - [Bogdan]



