# Lesson 14.

## 1. Часть первая

### 1.1 Внутри ВМ создать 2 контейнера LXD с именами node1 и node2

```sh
root@ubuntu:/home/ivan# lxc list
+-------+---------+-----------------------+------+-----------+-----------+
| NAME  |  STATE  |         IPV4          | IPV6 |   TYPE    | SNAPSHOTS |
+-------+---------+-----------------------+------+-----------+-----------+
| node1 | RUNNING | 10.103.253.196 (eth0) |      | CONTAINER | 0         |
+-------+---------+-----------------------+------+-----------+-----------+
| node2 | RUNNING | 10.103.253.65 (eth0)  |      | CONTAINER | 0         |
+-------+---------+-----------------------+------+-----------+-----------+

```
### 1.2 Обеспечить доступ по ssh-ключам

```sh
root@node1:~# ssh node2
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-58-generic x86_64)
root@node2:~# ssh node1
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-58-generic x86_64)

```
### 1.3 Создать в домашних папках папку backup_logs и выполнить разовую синхронизацию папки /var/log/apt друг на друга в папку backup_logs исключив файл term.log

```sh

root@node1:~# rsync -ra --exclude 'term.log' node2:/var/log/apt/ backup_logs/
root@node1:~# ls backup_logs/
eipp.log.xz  history.log

root@node2:~# rsync -ra --exclude 'term.log' node1:/var/log/apt/ backup_logs/
root@node2:~# ls backup_logs/
eipp.log.xz  history.log

```
### 1.4 Установить rsyncd и создать группы настроек для папки/var/log/apt

```sh
root@node1:~# nano /etc/rsyncd.conf
root@node1:~# iptables -I INPUT 1 -p tcp --dport 873 -j ACCEPT
root@node1:~# iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT
root@node2:~# cat /etc/rsyncd.conf
uid = root
gid = root
exclude = term.log
read only = no
[data1]
path = /var/log/apt/
comment = log
hosts allow = 10.103.253.196
root@node1:~# systemctl restart rsync
На  втором серевере тоже самое проделываем тоже самое
Проверяем
root@node1:~# rsync -rva --exclude 'term.log' node2::data1 backup_logs/
receiving incremental file list

sent 34 bytes  received 93 bytes  254.00 bytes/sec
total size is 25,144  speedup is 197.98
root@node1:~#

```
### 1.5 Настроить синхронизацию папки /var/log/apt в кроне с использованием rsyncd исключив файл term.log

```sh

root@node1:~# journalctl -f
Feb 01 09:58:01 node1 CRON[2174]: pam_unix(cron:session): session opened for user root(uid=0) by (uid=0)
Feb 01 09:58:01 node1 CRON[2175]: (root) CMD (rsync -ra --exclude 'term.log' node2::data1 backup_logs/)
Feb 01 09:58:01 node1 CRON[2174]: pam_unix(cron:session): session closed for user root

```
### 1.6 Настроить синхронизацию папки /var/log/apt c помощью systemd timers с использованием rsyncd исключив файл term.log

```sh
root@node1:~# cat /etc/systemd/system/my_rsyncd.service
[Unit]
Description=sync logs
After=network.target
[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -av node2::data1 /root/backup_logs/
[Install]
WantedBy=multi-user.target

root@node1:~# cat /etc/systemd/system/timer_for_sync.timer
[Unit]
Description=timer for rsync
Requires=my_rsyncd.service
[Timer]
Unit=my_rsyncd.service
OnCalendar=*:30
[Install]
WantedBy=timers.target

root@node1:~# systemctl enable timer_for_sync.timer
Created symlink /etc/systemd/system/timers.target.wants/timer_for_sync.timer → /etc/systemd/system/timer_for_sync.timer.
root@node1:~# journalctl -f
Feb 01 10:26:31 node1 systemd[1]: Starting sync logs...
Feb 01 10:26:31 node1 rsync[2371]: receiving incremental file list
Feb 01 10:26:31 node1 rsync[2372]: sent 20 bytes  received 93 bytes  226.00 bytes/sec
Feb 01 10:26:31 node1 rsync[2372]: total size is 25,144  speedup is 222.51
Feb 01 10:26:31 node1 systemd[1]: my_rsyncd.service: Deactivated successfully.
Feb 01 10:26:31 node1 systemd[1]: Finished sync logs.

```
## 2. Часть вторая

### 2.1 Выведите все номера телефонов.

```sh
root@ubuntu:/home/ivan# awk -F: '{print $2}' text.txt
(510) 548-1278
(408) 538-2358
(206) 654-6279
(206) 548-1348
(206) 548-1278
(916) 343-6410
(406) 298-7744
(206) 548-1278
(916) 348-4278
(510) 548-5258
(408) 926-3456
(916) 440-1763
```
### 2.2 Выведите номер телефона, принадлежащий сотрудника Dan.

```sh
Просто
root@ubuntu:/home/ivan# awk '/^Dan/' text.txt  | awk -F: '{print $2}'
(406) 298-7744
Сложно
oot@ubuntu:/home/ivan# awk -F: '{if ($1 = /^Dan/) print  $2}' text.txt
(406) 298-7744
Еще сложнее
root@ubuntu:/home/ivan# awk -F: 'BEGIN {print "Number of Dan: "}{if ($1 = /^Dan/) print  $2}' text.txt               
Number of Dan:
(406) 298-7744


```
### 2.3 Выведите имя, фамилию и номер телефона сотрудницы Susan.

```sh
root@ubuntu:/home/ivan# awk '/^Susan/' text.txt  | awk -F: '{print $1,$2}'
Susan Dalsass (206) 654-6279
ivan@ubuntu:~$ awk -F: '{if ($1 ~ /^Susan/) print $1,$2}' text.txt
Susan Dalsass (206) 654-6279

```
### 2.4 Выведите все фамилии, начинающиеся с буквы D.

```sh
root@ubuntu:/home/ivan# awk  '{if ($2 ~ /^D/) print $2}' text.txt | awk -F: '{print $1}'
Dobbins
Dalsass

Если делать без второй части получится 
root@ubuntu:/home/ivan# awk  '{if ($2 ~ /^D/) print $2}' text.txt
Dobbins:(408)
Dalsass:(206)
Как получить нужный результат одной командой я не придумал
```
### 2.5 Выведите все имена, начинающиеся с буквы C или E.

```sh
root@ubuntu:/home/ivan# awk  '{if ($1 ~ /^C/ || $1 ~/^E/) print $1}' text.txt
Christian
Chet
Elizabeth

```
### 2.6 Выведите все имена, состоящие только из четырех букв.

```sh
root@ubuntu:/home/ivan# awk 'length ($1) == 4 {print $1}' text.txt
Mike
Jody
John
Chet

```
### 2.7 Выведите имена сотрудников, префикс номера телефона которых 916.

```sh
root@ubuntu:/home/ivan# awk -F: '{if ($2 ~ /^\(916/) print $1}' text.txt
Guy Quigley
John Goldenrod
Elizabeth Stachelin

Если только имена, то по простому
root@ubuntu:/home/ivan# awk -F: '{if ($2 ~ /^\(916/) print $1}' text.txt | awk '{print $1}'
Guy
John
Elizabeth

И по сложному
ivan@ubuntu:~$ awk 'NR <=2 {next} $2~(916) {print $1}' text.txt
Guy
John
Elizabeth

```
### 2.8 Выведите денежные вклады сотрудника Mike, предваряя каждую сумму знаком $

```sh
ivan@ubuntu:~$ awk -F: '{if ($1 ~ /^Mike/) print "$"$3,"$"$4,"$"$5}' text.txt   
$250 $100 $175

```1234
