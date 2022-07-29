## Описание файлов в директории
logFileFull.log - полный лог выполнения  

Vagrant_folder - все что понадобится для поднятия ВМ и краткое описание файлов в ней  
Vagrantfile - вагрант файл  
provisioning - директория с файлами провижинга  
```
provisioning/
├── defaults
│   └── main.yml
├── hosts
├── playbook.yml
└── template
    ├── daemons
    └── frr.conf.j2
```

## Как выглядит сеть
![net](/net.jpg "Сеть дз")

## Описание как запустить виртуальную машину (кратко)
```
vagrant up
vagrant ssh router1
root@router1:~# ping -I 192.168.10.1 192.168.20.1

vagrant ssh router1
tcpdump -i enp0s9
tcpdump -i enp0s8
```

## ansible playbook только важное
Получилось обойтись только одним playbook, ниже важные опции
### provisioning/defaults/main.yml
Задаем две переменные, которые нам понадобятся далее и создал словарь и поместил в него те значения которые менялись в конфигурационном файле
```
router_id_enable: false
symmetric_routing: true

frr_var:
  - name: router1
    network1: '10.0.10.0/30'
    network2: '10.0.12.0/30'
    network3: '192.168.10.0/24'
    neighbor1: '10.0.10.2'
    neighbor2: '10.0.12.2'
    enp0s8: '10.0.10.1/30'
    enp0s8des: 'r1-r2'
    enp0s9: '10.0.12.1/30'
    enp0s9des: 'r1-r3'
    enp0s10: '192.168.10.1/24'
    enp0s10des: 'net_router1'
  - name: router2
    network1: '10.0.10.0/30'
    network2: '10.0.11.0/30'
    network3: '192.168.20.0/24'
    neighbor1: '10.0.10.1'
    neighbor2: '10.0.11.1'
    enp0s8: '10.0.10.2/30'
    enp0s8des: 'r2-r1'
    enp0s9: '10.0.11.2/30'
    enp0s9des: 'r2-r3'
    enp0s10: '192.168.20.1/24'
    enp0s10des: 'net_router2'
  - name: router3
    network1: '10.0.11.0/30'
    network2: '10.0.12.0/30'
    network3: '192.168.30.0/24'
    neighbor1: '10.0.11.2'
    neighbor2: '10.0.12.1'
    enp0s8: '10.0.11.1/30'
    enp0s8des: 'r3-r2'
    enp0s9: '10.0.12.2/30'
    enp0s9des: 'r3-r1'
    enp0s10: '192.168.30.1/24'
    enp0s10des: 'net_router3'

```
### provisioning/template/frr.conf.j2
Далее через цикл, с использованием условия выполняем соответствие имени ВМ и выбор нужных переменных
```
interface enp0s8
{% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
{% for router in frr_var %}
{% if ansible_hostname == router.name %}
 description {{ router.enp0s8des }}
 ip address {{ router.enp0s8 }}
{% endif %}
{% endfor %}
....
router ospf
{% if router_id_enable == false %}!{% endif %}router-id {{ router_id }}
{% for router in frr_var %}
{% if ansible_hostname == router.name %}
 network {{ router.network1 }} area 0
 network {{ router.network2 }} area 0
 network {{ router.network3 }} area 0
 neighbor {{ router.neighbor1 }}
 neighbor {{ router.neighbor2 }}
{% endif %}
{% endfor %}
```
Увеличиваем стоимость маршрута через интерфейс enp0s8, чтобы получить ассиметричный роутинг. Так же есть дополнительная проверка переменной symmetric_routing которую мы указали ранее, с помощью нее мы включим симметричный роутинг.
```
{% if ansible_hostname == 'router1' %}
 ip ospf cost 1000
{% elif ansible_hostname == 'router2' and symmetric_routing == true %}
```

## Логи
### Лог ассиметричного роутинга
Подключаемся к router1. С помощью комманды traceroute мы можем убедиться что маршрут изменился. Запускаем ping До router2
```
root@router1:~# traceroute 192.168.20.1
traceroute to 192.168.20.1 (192.168.20.1), 30 hops max, 60 byte packets
 1  10.0.12.2 (10.0.12.2)  0.578 ms  0.807 ms  0.670 ms
 2  192.168.20.1 (192.168.20.1)  1.015 ms  1.192 ms  0.873 ms
root@router1:~# ping -I 192.168.10.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=1.20 ms
...
```
Подключаемся к router2 и запускаем tcpdump на нужных интерфейсах. И видим что получаем пакеты мы с одного интерфейса а отправляем через другой.
```
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
12:23:12.922816 IP 192.168.10.1 > router2: ICMP echo request, id 1, seq 14, length 64
12:23:13.924262 IP 192.168.10.1 > router2: ICMP echo request, id 1, seq 15, length 64
12:23:14.926212 IP 192.168.10.1 > router2: ICMP echo request, id 1, seq 16, length 64
12:23:15.927994 IP 192.168.10.1 > router2: ICMP echo request, id 1, seq 17, length 64
...
25 packets captured
25 packets received by filter
0 packets dropped by kernel
root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
12:23:35.065902 IP router2 > 192.168.10.1: ICMP echo reply, id 1, seq 36, length 64
12:23:36.067684 IP router2 > 192.168.10.1: ICMP echo reply, id 1, seq 37, length 64
...
7 packets captured
7 packets received by filter
0 packets dropped by kernel
```

### Лог восстановление симетричного роутинга
Изменяем переменную symmetric_routing: true и выполняем провижинг.  
Подключаемся к router2 и снимаем дамп
```
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
13:03:54.177989 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 14, length 64
13:03:54.178030 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 14, length 64
13:03:55.179311 IP 192.168.10.1 > router2: ICMP echo request, id 2, seq 15, length 64
13:03:55.179360 IP router2 > 192.168.10.1: ICMP echo reply, id 2, seq 15, length 64
...
11 packets captured
11 packets received by filter
0 packets dropped by kernel
```

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)
