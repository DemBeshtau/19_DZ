# Установка и настройка DHCP, TFTP и PXE-загрузкчика.
1. Настроить загрузку по сети дистрибутива Ubuntu 24;
2. Установка производится из HTTP-репозитория;
3. Настроить автоматическую установку с помощью файла user-data.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0 и образа ubuntu/jammy64 версии 20240301.0.0. <br/> 
### Ход решения ###
&ensp;&ensp;Для получения клиентом ip-адреса требуется DHCP-сервер, для получения файла pxelinux.0 необходима установка   TFTP-сервера. Для решения указанных задач необходимо установить утилиту dnsmasq, совмещающую в себе сразу оба сервера.
1. Отключение файервола:
```shell
root@pxeserver:~# systemctl stop ufw
root@pxeserver:~# systemctl disable ufw
root@pxeserver:~# systemctl status ufw
○ ufw.service - Uncomplicated firewall
     Loaded: loaded (/lib/systemd/system/ufw.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: man:ufw(8)

Jul 14 13:12:33 ubuntu systemd[1]: Starting Uncomplicated firewall...
Jul 14 13:12:33 ubuntu systemd[1]: Finished Uncomplicated firewall.
Jul 14 13:12:56 pxeserver systemd[1]: Stopping Uncomplicated firewall...
Jul 14 13:12:56 pxeserver ufw-init[2104]: Skip stopping firewall: ufw (not enabled)
Jul 14 13:12:56 pxeserver systemd[1]: ufw.service: Deactivated successfully.
Jul 14 13:12:56 pxeserver systemd[1]: Stopped Uncomplicated firewall.
```
2. Установка утилиты dnsmasq:
```shell
root@pxeserver:~# apt update && apt install dnsmasq -y
...
```
3. Подготовка конфигурационного файла /etc/dnsmasq.d/pxe.conf:
```shell
root@pxeserver:~# nano /etc/dnsmasq.d/pxe.conf
...
root@pxeserver:~# cat /etc/dnsmasq.d/pxe.conf
interface=enp0s8
bind-interfaces
dhcp-range=enp0s8, 10.0.0.100, 10.0.0.120
dhcp-boot=pxelinux.0
enable-tftp
tftp-root=/srv/tftp/amd64
```
4. Создание каталога для TFTP-сервера, скачивание файла для сетевой установки и его распаковка:
```shell
root@pxeserver:~# mkdir /srv/tftp
root@pxeserver:~# wget https://mirror.yandex.ru/ubuntu-releases/24.04/ubuntu-24.04-netboot-amd64.tar.gz
root@pxeserver:~# tar -xzvf ubuntu-24.04-netboot-amd64.tar.gz -C /srv/ftp
root@pxeserver:~# ll /srv/tftp/amd64
total 85268
drwxr-xr-x 4 root root     4096 Apr 23 09:46 ./
drwxr-xr-x 3 root root     4096 Apr 23 09:46 ../
-rw-r--r-- 1 root root   966664 Apr  4 12:39 bootx64.efi
drwxr-xr-x 2 root root     4096 Apr 23 09:46 grub/
-rw-r--r-- 1 root root  2340744 Apr  4 10:24 grubx64.efi
-rw-r--r-- 1 root root 68889068 Apr 23 09:46 initrd
-rw-r--r-- 1 root root   118676 Apr  8 16:20 ldlinux.c32
-rw-r--r-- 1 root root 14928264 Apr 23 09:46 linux
-rw-r--r-- 1 root root    42392 Apr  8 16:20 pxelinux.0
drwxr-xr-x 2 root root     4096 Jul 14 15:14 pxelinux.cfg/
```
5. Установка и web-сервера apache2, необходимого для отдачи файлов по HTTP:
```shell
root@pxeserver:~# apt install apache2
...
```
6. Скачивание образа Ubuntu 24:
```shell
root@pxeserver:~# mkdir /srv/images && cd /srv/images
root@pxeserver:/srv/images# wget https://mirror.yandex.ru/ubuntu-releases/24.04/ubuntu-24.04-live-server-amd64.iso   
```
7. Подготовка страницы доступа к серверу TFTP:
```shell
root@pxeserver:~# nano /etc/apache2/sites-available/ks-server.conf
...
root@pxeserver:~# cat /etc/apache2/sites-available/ks-server.conf
<VirtualHost 10.0.0.20:80>
DocumentRoot /
	<Directory /srv/ks>
		Options Indexes MultiViews
		AllowOverride All
		Require all granted
	</Directory>
	<Directory /srv/images>
		Options Indexes MultiViews
		AllowOverride All
		Require all granted	
	</Directory>
</VirtualHost>
```
8. Активирование конфигурации ks-server.conf в apache:
```shell
root@pxeserver:/etc/apache2/sites-available/# a2ensite ks-server.conf
```
9. Подготовка файла конфигурации загрузчика PXE /srv/tftp/amd64/pxelinux.cfg/default:
```shell
root@pxeserver:~# nano /srv/tftp/amd64/pxelinux.cfg/default
...
root@pxeserver:~# cat /srv/tftp/amd64/pxelinux.cfg/default
DEFAULT install
LABEL install
  KERNEL linux
  INITRD initrd
  APPEND root=/dev/ram0 ramdisk_size=3000000 ip=dhcp iso-url=http://10.0.0.20/srv/images/ubuntu-24.04-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://10.0.0.20/srv/ks
```
10. Подготовка файла для автоматизированной установки Ubuntu 24:
```shell
root@pxeserver:~# mkdir /srv/ks
root@pxeserver:~# nano /srv/ks/user-data
...
root@pxeserver:~# cat /srv/ks/user-data
#cloud-config
autoinstall:
  apt:
    disable_components: []
    geoip: true
    preserve_sources_list: false
    primary:
      - arches:
          - amd64
          - i386
        uri: http://us.archive.ubuntu.com/ubuntu
      - arches:
          - default
        uri: http://ports.ubuntu.com/ubuntu-ports
  drivers:
    install: false
  identity:
    hostname: linux
    password: $6$sJgo6Hg5zXBwkkI8$btrEoWAb5FxKhajagWR49XM4EAOfO/Dr5bMrLOkGe3KkMYdsh7T3MU5mYwY2TIMJpVKckAwnZFs2ltUJ1abOZ.
    realname: otus
    username: otus
  kernel:
    package: linux-generic
  keyboard:
    layout: us
    toggle: null
    variant: ''
  locale: en_US.UTF-8
  network:
    version: 2
    ethernets:
      enp0s3:
        dhcp4: true
      enp0s8:
        dhcp4: true
  ssh:
    allow-pw: true
    authorized-keys: []
    install-server: true
  updates: security
  version: 1
```
11. Перезапуск сервисов:
```shell
root@pxeserver:~# systemctl restart dnsmasq
root@pxeserver:~# systemctl restart apache2
```
12. После запуска ВМ pxeclient произойдёт автоматическая настройка её сетевого интерфейса по DHCP и далее стартует установка ОС Ubuntu 24. После установки ОС, необходимо выключить ВМ и, либо в её настройках поставить запуск с диска, либо при повторном запуске по F12 выбрать загрузочный диск и запустить систему.
![изображение](https://github.com/user-attachments/assets/ce52422f-4093-45f3-bd66-5cae3c55d1e6)

