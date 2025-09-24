# 1. Установка пакетов
```bash
apt-get update
apt-get install openvpn easy-rsa
```
# 2. Подготовка инфраструктуры ключей
```bash
mkdir /etc/openvpn/easy-rsa
cp -r /usr/share/easyrsa3/* /etc/openvpn/easy-rsa/
cd /etc/openvpn/easy-rsa
easyrsa init-pki
```
# 3. Генерация корневого сертификата (CA)
```bash
easyrsa build-ca
```
# 4. Генерация серверных сертификатов
```bash
easyrsa gen-req server nopass
easyrsa sign-req server server
```
# 5. Генерация клиентских сертификатов
```bash
easyrsa gen-req client nopass
easyrsa sign-req client client
```
# 6. Генерация Diffie-Hellman параметров
```bash
easyrsa gen-dh
```
# 7. Копирование файлов в нужные директории
```bash
cp pki/ca.crt pki/private/server.key pki/issued/server.crt pki/dh.pem /etc/openvpn/
```
# 8. Базовая конфигурация сервера
Создайте /etc/openvpn/server.conf со следующим содержимым:
```ini
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
keepalive 10 120
persist-key
persist-tun
status openvpn-status.log
verb 3
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 77.88.8.8"
```
# 9. Включение форвардинга и настройка фаервола
Правим `net.ipv4.ip_forward=1` в `/etc/net/sysctl.conf`. Потом:
```bash
sysctl -p
```
Заменяем XXX на свой внешний интерфейс, например enp2s0.
```bash
iptables -A INPUT -p udp --dport 1194 -j ACCEPT
iptables -A FORWARD -i tun0 -j ACCEPT
iptables -A FORWARD -i tun0 -o XXX -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i XXX -o tun0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o XXX -j MASQUERADE

iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```
# 10. Запуск службы
Для запуска сервера используется скрипт `/etc/openvpn/openvpn-startup`. Впишите в него указание для `openvpn` запускать сервер с конфигурации `server.conf` в фоне:
```bash
openvpn /etc/openvpn/server.conf &
```
В результате файл `/etc/openvpn/openvpn-startup` должен иметь вид:
```bash
#!/bin/bash
# Startup file for OpenVPN
openvpn /etc/openvpn/server.conf &
# Load tun module
/sbin/modprobe tun >/dev/null 2>&1
sleep 1s
```
Добавьте службу в автозапуск:
```bash
systemctl enable openvpn
```
Запустите сервер:
```bash
systemctl start openvpn
```
