# WEB. High Performance WEB и CDN

# Задание 1. Создание Server Pool для backend

Запустим HTTP-серверы на Python:

```bash
root@hiplet-48656:~# mkdir -p ~/backend1 ~/backend2
root@hiplet-48656:~# echo "<h1>Response from Backend Server 1</h1>" > ~/backend1/index.html
root@hiplet-48656:~# echo "<h2>*** Response from Backend Server 2 ***</h2>" > ~/backend2/index.html
root@hiplet-48656:~# cd ~/backend1 && python3 -m http.server 8081 > /dev/null 2>&1 &
[1] 693361
root@hiplet-48656:~# cd ~/backend2 && python3 -m http.server 8082 > /dev/null 2>&1 &
[2] 693369
root@hiplet-48656:~# curl http://127.0.0.1:8081
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://127.0.0.1:8082
<h2>*** Response from Backend Server 2 ***</h2>
root@hiplet-48656:~# 
```

# Задание 2. DNS Load Balancing с помощью dnsmasq

Настроим и запустим dnsmasq:

```bash
root@hiplet-48656:~# mkdir -p /etc/dnsmasq.d
root@hiplet-48656:~# cat > /etc/dnsmasq.d/loadbalancing.conf << 'EOF'
address=/my-awesome-highload-app.local/127.0.0.1
address=/my-awesome-highload-app.local/127.0.0.2
EOF
root@hiplet-48656:~# cat > /etc/dnsmasq.conf << 'EOF'
port=53
no-resolv
conf-dir=/etc/dnsmasq.d
root@hiplet-48656:~# systemctl start dnsmasq
EOF
```

Проверим, что он отдаёт:
```bash
root@hiplet-48656:~# dig @127.0.0.1 my-awesome-highload-app.local

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> @127.0.0.1 my-awesome-highload-app.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 26468
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;my-awesome-highload-app.local. IN      A

;; ANSWER SECTION:
my-awesome-highload-app.local. 0 IN     A       127.0.0.2
my-awesome-highload-app.local. 0 IN     A       127.0.0.1

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Thu Oct 30 09:52:51 UTC 2025
;; MSG SIZE  rcvd: 90

root@hiplet-48656:~# 
```

DNS-сервер dnsmasq не проверяет доступность backend-серверов, с протоколом HTTP он никак не связан. Даже если backend2 (127.0.0.2:8082) сломается, dnsmasq продолжит возвращать обе A-записи.
Никакого health-check механизма, здесь нет, клиенты будут получать оба IP-адреса и могут попытаться подключиться к недоступному серверу.

После убийства обоих серверов вывод такой же:
```bash
root@hiplet-48656:~# dig @127.0.0.1 my-awesome-highload-app.local +short
127.0.0.2
127.0.0.1
```

# Задание 3. Балансировка Layer 4 с помощью IPVS

Создадим и поднимем dummy1 интерфейс:
```bash
root@hiplet-48656:~# modprobe dummy
root@hiplet-48656:~# ip link add dummy1 type dummy
root@hiplet-48656:~# ip link set dummy1 up
root@hiplet-48656:~# ip addr add 192.168.100.1/32 dev dummy1
root@hiplet-48656:~# ip addr show dummy1
10: dummy1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether de:91:04:04:b1:f5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/32 scope global dummy1
       valid_lft forever preferred_lft forever
    inet6 fe80::dc91:4ff:fe04:b1f5/64 scope link 
       valid_lft forever preferred_lft forever
root@hiplet-48656:~# 
```

Загрузим модули ядра для IPVS:
```bash
root@hiplet-48656:~# modprobe ip_vs
root@hiplet-48656:~# 
root@hiplet-48656:~# modprobe ip_vs_rr
root@hiplet-48656:~# 
root@hiplet-48656:~# modprobe ip_vs_wrr
root@hiplet-48656:~# 
root@hiplet-48656:~# modprobe ip_vs_sh
root@hiplet-48656:~# 
```

Настроим правила IPVS с round-robin балансировкой:
```bash
root@hiplet-48656:~# ipvsadm -C
root@hiplet-48656:~# ipvsadm -A -t 192.168.100.1:80 -s rr
root@hiplet-48656:~# ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m
root@hiplet-48656:~# ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m
root@hiplet-48656:~# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0         
  -> 127.0.0.1:8082               Masq    1      0          0         
root@hiplet-48656:~# 
```

Проверим, что round-robin отрабатывает и ответы попеременно меняются:
```bash
root@hiplet-48656:~# curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
root@hiplet-48656:~# curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
root@hiplet-48656:~# curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://192.168.100.1
<h2>*** Response from Backend Server 2 ***</h2>
root@hiplet-48656:~# curl http://192.168.100.1
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# 
```

Посмотрим на статистику IPVS:
```bash
root@hiplet-48656:~# ipvsadm -L -n --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  192.168.100.1:80                    6       38       34     2480     3190
  -> 127.0.0.1:8081                      3       20       16     1292     1531
  -> 127.0.0.1:8082                      3       18       18     1188     1659
root@hiplet-48656:~# 
```

# Задание 4. Балансировка L7 с помощью NGINX

Создадим nginx конфиг и проверим, что он валидный:
```bash
root@hiplet-48656:~# cat > /etc/nginx/conf.d/highload-backend.conf << 'EOF'
upstream backend_pool {
    server 127.0.0.1:8081 max_fails=7 fail_timeout=30s;
    server 127.0.0.1:8082 backup;
}

server {
    listen 127.0.0.1:8888;
    
    location / {
        proxy_pass http://backend_pool;
        proxy_set_header X-high-load-test "123";
        proxy_set_header Host $host;
        proxy_next_upstream off;
    }
}
EOF
root@hiplet-48656:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
root@hiplet-48656:~# 
```

Проверим, что nginx слушает 8888 порт:
```bash
root@hiplet-48656:~# nginx -s reload
2025/10/30 10:08:33 [notice] 694737#694737: signal process started
root@hiplet-48656:~# ss -tuln | grep 8888
tcp   LISTEN 0      511             127.0.0.1:8888      0.0.0.0:*          
root@hiplet-48656:~# 
```

Делаем запросы и видим, что отвечает первый сервер:
```bash
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# 
```

Теперь убьём первый сервер и увидим, что только через 7 попыток нам ответит второй (как и надо по заданию):
```bash
root@hiplet-48656:~# kill $(lsof -t -i:8081)
root@hiplet-48656:~# Terminated
kill $(lsof -t -i:8081)^C
[3]+  Exit 143                cd ~/backend1 && python3 -m http.server 8081 > /dev/null 2>&1
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h2>*** Response from Backend Server 2 ***</h2>
root@hiplet-48656:~# 
```

Без настройки `proxy_next_upstream off;` в конфиге nginx, несмотря на max_fails, nginx сразу перенаправлял запрос на второй сервер.

Запустим сервер обратно.

Теперь перехватим трафик через tshark:

```bash
root@hiplet-48656:~# tshark -i lo -f "tcp port 8888 or tcp port 8081" -Y "http" -V > /tmp/tshark_capture.txt 2>&1 &
[4] 695285
root@hiplet-48656:~# cat /tmp/tshark_capture.txt
Running as user "root" and group "root". This could be dangerous.
Capturing on 'Loopback: lo'
root@hiplet-48656:~# curl http://127.0.0.1:8888
<h1>Response from Backend Server 1</h1>
root@hiplet-48656:~# kill -9 695285
root@hiplet-48656:~# cat /tmp/tshark_capture.txt
Running as user "root" and group "root". This could be dangerous.
Capturing on 'Loopback: lo'
Frame 4: 143 bytes on wire (1144 bits), 143 bytes captured (1144 bits) on interface lo, id 0
    Section number: 1
    Interface id: 0 (lo)
        Interface name: lo
    Encapsulation type: Ethernet (1)
    Arrival Time: Oct 30, 2025 10:20:14.145975649 UTC
    UTC Arrival Time: Oct 30, 2025 10:20:14.145975649 UTC
    Epoch Arrival Time: 1761819614.145975649
    [Time shift for this packet: 0.000000000 seconds]
    [Time delta from previous captured frame: 0.000133549 seconds]
    [Time delta from previous displayed frame: 0.000000000 seconds]
    [Time since reference or first frame: 0.000208605 seconds]
    Frame Number: 4
    Frame Length: 143 bytes (1144 bits)
    Capture Length: 143 bytes (1144 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:tcp:http]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
    Destination: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 129
    Identification: 0x7f86 (32646)
    010. .... = Flags: 0x2, Don't fragment
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    ...0 0000 0000 0000 = Fragment Offset: 0
    Time to Live: 64
    Protocol: TCP (6)
    Header Checksum: 0xbcee [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 127.0.0.1
    Destination Address: 127.0.0.1
Transmission Control Protocol, Src Port: 57396, Dst Port: 8888, Seq: 1, Ack: 1, Len: 77
    Source Port: 57396
    Destination Port: 8888
    [Stream index: 0]
    [Conversation completeness: Incomplete, ESTABLISHED (7)]
        ..0. .... = RST: Absent
        ...0 .... = FIN: Absent
        .... 0... = Data: Absent
        .... .1.. = ACK: Present
        .... ..1. = SYN-ACK: Present
        .... ...1 = SYN: Present
        [Completeness Flags: ···ASS]
    [TCP Segment Len: 77]
    Sequence Number: 1    (relative sequence number)
    Sequence Number (raw): 2266104552
    [Next Sequence Number: 78    (relative sequence number)]
    Acknowledgment Number: 1    (relative ack number)
    Acknowledgment number (raw): 3093461617
    1000 .... = Header Length: 32 bytes (8)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Accurate ECN: Not set
        .... 0... .... = Congestion Window Reduced: Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······AP···]
    Window: 260
    [Calculated window size: 33280]
    [Window size scaling factor: 128]
    Checksum: 0xfe75 [unverified]
    [Checksum Status: Unverified]
    Urgent Pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 1519250213: TSval 1519250213, TSecr 1519250213
            Timestamp echo reply: 1519250213
    [Timestamps]
        [Time since first frame in this TCP stream: 0.000208605 seconds]
        [Time since previous frame in this TCP stream: 0.000133549 seconds]
    [SEQ/ACK analysis]
        [iRTT: 0.000075056 seconds]
        [Bytes in flight: 77]
        [Bytes sent since last PSH flag: 77]
    TCP payload (77 bytes)
Hypertext Transfer Protocol
    GET / HTTP/1.1\r\n
        [Expert Info (Chat/Sequence): GET / HTTP/1.1\r\n]
            [GET / HTTP/1.1\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.1
    Host: 127.0.0.1:8888\r\n
    User-Agent: curl/8.5.0\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://127.0.0.1:8888/]
    [HTTP request 1/1]

Frame 9: 180 bytes on wire (1440 bits), 180 bytes captured (1440 bits) on interface lo, id 0
    Section number: 1
    Interface id: 0 (lo)
        Interface name: lo
    Encapsulation type: Ethernet (1)
    Arrival Time: Oct 30, 2025 10:20:14.146194569 UTC
    UTC Arrival Time: Oct 30, 2025 10:20:14.146194569 UTC
    Epoch Arrival Time: 1761819614.146194569
    [Time shift for this packet: 0.000000000 seconds]
    [Time delta from previous captured frame: 0.000036970 seconds]
    [Time delta from previous displayed frame: 0.000218920 seconds]
    [Time since reference or first frame: 0.000427525 seconds]
    Frame Number: 9
    Frame Length: 180 bytes (1440 bits)
    Capture Length: 180 bytes (1440 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:tcp:http]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
    Destination: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 166
    Identification: 0xfc8d (64653)
    010. .... = Flags: 0x2, Don't fragment
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    ...0 0000 0000 0000 = Fragment Offset: 0
    Time to Live: 64
    Protocol: TCP (6)
    Header Checksum: 0x3fc2 [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 127.0.0.1
    Destination Address: 127.0.0.1
Transmission Control Protocol, Src Port: 54224, Dst Port: 8081, Seq: 1, Ack: 1, Len: 114
    Source Port: 54224
    Destination Port: 8081
    [Stream index: 1]
    [Conversation completeness: Incomplete, ESTABLISHED (7)]
        ..0. .... = RST: Absent
        ...0 .... = FIN: Absent
        .... 0... = Data: Absent
        .... .1.. = ACK: Present
        .... ..1. = SYN-ACK: Present
        .... ...1 = SYN: Present
        [Completeness Flags: ···ASS]
    [TCP Segment Len: 114]
    Sequence Number: 1    (relative sequence number)
    Sequence Number (raw): 670890712
    [Next Sequence Number: 115    (relative sequence number)]
    Acknowledgment Number: 1    (relative ack number)
    Acknowledgment number (raw): 211452958
    1000 .... = Header Length: 32 bytes (8)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Accurate ECN: Not set
        .... 0... .... = Congestion Window Reduced: Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······AP···]
    Window: 260
    [Calculated window size: 33280]
    [Window size scaling factor: 128]
    Checksum: 0xfe9a [unverified]
    [Checksum Status: Unverified]
    Urgent Pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 1519250213: TSval 1519250213, TSecr 1519250213
            Timestamp echo reply: 1519250213
    [Timestamps]
        [Time since first frame in this TCP stream: 0.000052491 seconds]
        [Time since previous frame in this TCP stream: 0.000036970 seconds]
    [SEQ/ACK analysis]
        [iRTT: 0.000015521 seconds]
        [Bytes in flight: 114]
        [Bytes sent since last PSH flag: 114]
    TCP payload (114 bytes)
Hypertext Transfer Protocol
    GET / HTTP/1.0\r\n
        [Expert Info (Chat/Sequence): GET / HTTP/1.0\r\n]
            [GET / HTTP/1.0\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Request Method: GET
        Request URI: /
        Request Version: HTTP/1.0
    X-high-load-test: 123\r\n
    Host: 127.0.0.1\r\n
    Connection: close\r\n
    User-Agent: curl/8.5.0\r\n
    Accept: */*\r\n
    \r\n
    [Full request URI: http://127.0.0.1/]
    [HTTP request 1/1]

Frame 13: 106 bytes on wire (848 bits), 106 bytes captured (848 bits) on interface lo, id 0
    Section number: 1
    Interface id: 0 (lo)
        Interface name: lo
    Encapsulation type: Ethernet (1)
    Arrival Time: Oct 30, 2025 10:20:14.151664400 UTC
    UTC Arrival Time: Oct 30, 2025 10:20:14.151664400 UTC
    Epoch Arrival Time: 1761819614.151664400
    [Time shift for this packet: 0.000000000 seconds]
    [Time delta from previous captured frame: 0.000134364 seconds]
    [Time delta from previous displayed frame: 0.005469831 seconds]
    [Time since reference or first frame: 0.005897356 seconds]
    Frame Number: 13
    Frame Length: 106 bytes (848 bits)
    Capture Length: 106 bytes (848 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:tcp:http:data-text-lines]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
    Destination: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 92
    Identification: 0x4543 (17731)
    010. .... = Flags: 0x2, Don't fragment
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    ...0 0000 0000 0000 = Fragment Offset: 0
    Time to Live: 64
    Protocol: TCP (6)
    Header Checksum: 0xf756 [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 127.0.0.1
    Destination Address: 127.0.0.1
Transmission Control Protocol, Src Port: 8081, Dst Port: 54224, Seq: 186, Ack: 115, Len: 40
    Source Port: 8081
    Destination Port: 54224
    [Stream index: 1]
    [Conversation completeness: Incomplete, DATA (15)]
        ..0. .... = RST: Absent
        ...0 .... = FIN: Absent
        .... 1... = Data: Present
        .... .1.. = ACK: Present
        .... ..1. = SYN-ACK: Present
        .... ...1 = SYN: Present
        [Completeness Flags: ··DASS]
    [TCP Segment Len: 40]
    Sequence Number: 186    (relative sequence number)
    Sequence Number (raw): 211453143
    [Next Sequence Number: 226    (relative sequence number)]
    Acknowledgment Number: 115    (relative ack number)
    Acknowledgment number (raw): 670890826
    1000 .... = Header Length: 32 bytes (8)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Accurate ECN: Not set
        .... 0... .... = Congestion Window Reduced: Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······AP···]
    Window: 260
    [Calculated window size: 33280]
    [Window size scaling factor: 128]
    Checksum: 0xfe50 [unverified]
    [Checksum Status: Unverified]
    Urgent Pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 1519250218: TSval 1519250218, TSecr 1519250218
            Timestamp echo reply: 1519250218
    [Timestamps]
        [Time since first frame in this TCP stream: 0.005522322 seconds]
        [Time since previous frame in this TCP stream: 0.000134364 seconds]
    [SEQ/ACK analysis]
        [iRTT: 0.000015521 seconds]
        [Bytes in flight: 40]
        [Bytes sent since last PSH flag: 40]
    TCP payload (40 bytes)
    TCP segment data (40 bytes)
[2 Reassembled TCP Segments (225 bytes): #11(185), #13(40)]
    [Frame: 11, payload: 0-184 (185 bytes)]
    [Frame: 13, payload: 185-224 (40 bytes)]
    [Segment count: 2]
    [Reassembled TCP length: 225]
    [Reassembled TCP Data [truncated]: 485454502f312e3020323030204f4b0d0a5365727665723a2053696d706c65485454502f302e3620507974686f6e2f332e31322e330d0a446174653a205468752c203330204f637420323032352031303a32303a313420474d540d0a436f6e74656e742d74797]
Hypertext Transfer Protocol
    HTTP/1.0 200 OK\r\n
        [Expert Info (Chat/Sequence): HTTP/1.0 200 OK\r\n]
            [HTTP/1.0 200 OK\r\n]
            [Severity level: Chat]
            [Group: Sequence]
        Response Version: HTTP/1.0
        Status Code: 200
        [Status Code Description: OK]
        Response Phrase: OK
    Server: SimpleHTTP/0.6 Python/3.12.3\r\n
    Date: Thu, 30 Oct 2025 10:20:14 GMT\r\n
    Content-type: text/html\r\n
    Content-Length: 40\r\n
        [Content length: 40]
    Last-Modified: Thu, 30 Oct 2025 09:43:31 GMT\r\n
    \r\n
    [HTTP response 1/1]
    [Time since request: 0.005469831 seconds]
    [Request in frame: 9]
    [Request URI: http://127.0.0.1/]
    File Data: 40 bytes
Line-based text data: text/html (1 lines)
    <h1>Response from Backend Server 1</h1>\n

Frame 18: 308 bytes on wire (2464 bits), 308 bytes captured (2464 bits) on interface lo, id 0
    Section number: 1
    Interface id: 0 (lo)
        Interface name: lo
    Encapsulation type: Ethernet (1)
    Arrival Time: Oct 30, 2025 10:20:14.151959469 UTC
    UTC Arrival Time: Oct 30, 2025 10:20:14.151959469 UTC
    Epoch Arrival Time: 1761819614.151959469
    [Time shift for this packet: 0.000000000 seconds]
    [Time delta from previous captured frame: 0.000026458 seconds]
    [Time delta from previous displayed frame: 0.000295069 seconds]
    [Time since reference or first frame: 0.006192425 seconds]
    Frame Number: 18
    Frame Length: 308 bytes (2464 bits)
    Capture Length: 308 bytes (2464 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:ethertype:ip:tcp:http:data-text-lines]
Ethernet II, Src: 00:00:00_00:00:00 (00:00:00:00:00:00), Dst: 00:00:00_00:00:00 (00:00:00:00:00:00)
    Destination: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:00_00:00:00 (00:00:00:00:00:00)
        Address: 00:00:00_00:00:00 (00:00:00:00:00:00)
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: IPv4 (0x0800)
Internet Protocol Version 4, Src: 127.0.0.1, Dst: 127.0.0.1
    0100 .... = Version: 4
    .... 0101 = Header Length: 20 bytes (5)
    Differentiated Services Field: 0x00 (DSCP: CS0, ECN: Not-ECT)
        0000 00.. = Differentiated Services Codepoint: Default (0)
        .... ..00 = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    Total Length: 294
    Identification: 0xa2f4 (41716)
    010. .... = Flags: 0x2, Don't fragment
        0... .... = Reserved bit: Not set
        .1.. .... = Don't fragment: Set
        ..0. .... = More fragments: Not set
    ...0 0000 0000 0000 = Fragment Offset: 0
    Time to Live: 64
    Protocol: TCP (6)
    Header Checksum: 0x98db [validation disabled]
    [Header checksum status: Unverified]
    Source Address: 127.0.0.1
    Destination Address: 127.0.0.1
Transmission Control Protocol, Src Port: 8888, Dst Port: 57396, Seq: 1, Ack: 78, Len: 242
    Source Port: 8888
    Destination Port: 57396
    [Stream index: 0]
    [Conversation completeness: Incomplete, DATA (15)]
        ..0. .... = RST: Absent
        ...0 .... = FIN: Absent
        .... 1... = Data: Present
        .... .1.. = ACK: Present
        .... ..1. = SYN-ACK: Present
        .... ...1 = SYN: Present
        [Completeness Flags: ··DASS]
    [TCP Segment Len: 242]
    Sequence Number: 1    (relative sequence number)
    Sequence Number (raw): 3093461617
    [Next Sequence Number: 243    (relative sequence number)]
    Acknowledgment Number: 78    (relative ack number)
    Acknowledgment number (raw): 2266104629
    1000 .... = Header Length: 32 bytes (8)
    Flags: 0x018 (PSH, ACK)
        000. .... .... = Reserved: Not set
        ...0 .... .... = Accurate ECN: Not set
        .... 0... .... = Congestion Window Reduced: Not set
        .... .0.. .... = ECN-Echo: Not set
        .... ..0. .... = Urgent: Not set
        .... ...1 .... = Acknowledgment: Set
        .... .... 1... = Push: Set
        .... .... .0.. = Reset: Not set
        .... .... ..0. = Syn: Not set
        .... .... ...0 = Fin: Not set
        [TCP Flags: ·······AP···]
    Window: 260
    [Calculated window size: 33280]
    [Window size scaling factor: 128]
    Checksum: 0xff1a [unverified]
    [Checksum Status: Unverified]
    Urgent Pointer: 0
    Options: (12 bytes), No-Operation (NOP), No-Operation (NOP), Timestamps
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - No-Operation (NOP)
            Kind: No-Operation (1)
        TCP Option - Timestamps
            Kind: Time Stamp Option (8)
            Length: 10
            Timestamp value: 1519250219: TSval 1519250219, TSecr 1519250213
            Timestamp echo reply: 1519250213
    [Timestamps]
        [Time since first frame in this TCP stream: 0.006192425 seconds]
        [Time since previous frame in this TCP stream: 0.005974845 seconds]
    [SEQ/ACK analysis]
        [iRTT: 0.000075056 seconds]
        [Bytes in flight: 242]
        [Bytes sent since last PSH flag: 242]
    TCP payload (24[4]+  Killed                  tshark -i lo -f "tcp port 8888 or tcp port 8081" -Y "http" -V > /tmp/tshark_capture.txt 2>&1
root@hiplet-48656:~# 
```

Можно увидеть, как запрос приходит на nginx, проксируется на сервер и при этом передаётся нужный нам заголовок `X-high-load-test`.


### Frame 4: запрос от клиента к nginx (порт 8888)
```
GET / HTTP/1.1
Host: 127.0.0.1:8888
User-Agent: curl/8.5.0
Accept: */*
```

### Frame 9: запрос от nginx к backend1 (порт 8081)
```
GET / HTTP/1.0
X-high-load-test: 123
Host: 127.0.0.1
Connection: close
User-Agent: curl/8.5.0
Accept: */*
```

### Frame 13: ответ от backend1 к nginx
```
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.12.3
Content-Length: 40

<h1>Response from Backend Server 1</h1>
```

### Frame 18: ответ от nginx к curl'у
nginx возвращает ответ клиенту (242 байта данных).

