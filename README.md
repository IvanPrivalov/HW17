# Borgbackup

## Задание

Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client

Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:

- Репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение
- Имя бекапа должно содержать информацию о времени снятия бекапа
- Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов
- Резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации.
- Написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение.

## Выполнение ДЗ

### Установка borgbackup на обоих хостах

Скачиваем бинарник и делаем его исполняемым:

```shell
[root@server ~]# curl -L https://github.com/borgbackup/borg/releases/download/1.1.15/borg-linux64 -o /usr/bin/borg
[root@server ~]# chmod +x /usr/bin/borg
```

На хосте backup создаем пользователя borg:

```shell
[root@backup ~]# useradd -m borg
```

На хосте server генерируем SSH-ключ и добавляем его в ~borg/.ssh/authorized_keys на хост backup:

```shell
[root@server ~]# ssh-keygen
...
[root@backup ~]# mkdir ~borg/.ssh && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDDXmwAbDnzO6p5Pnxm0kgbGDTOX4ob9E94IFhmRmV/gqBikWzo3Jbms7nHR68CBjZGWyKchxzs+7mqZp/ynnQEfcrUc8dLGsDekQEgoSAVXDSJBWs6YuCgEPA7WewgMj3Pz0Xb+pCAFGf/ceZbGqPPlcvw0+olAeCSU+255ditZt/rlwV0NuloG2dKYR5QIxvVBvbVQ0RiXERb4KXNjgoCiPLPtPo4yWcweOwB0z07CeA43/lP75aooP0+erfiHDUjPK5b7l+DFTYol/l+zXyyT0eNUp7X5ocSLDiytJXlwymzIughc/TCKsVUg34IVDf07VgKN/62cq3DuWAPcYTl root@server" > ~borg/.ssh/authorized_keys
[root@backup ~]# chown -R borg:borg ~borg/.ssh
```

Проверяем подключение:

<details><summary>Получим следующий вывод</summary>
<p>

```log
[root@server .ssh]# ssh borg@192.168.10.20
The authenticity of host '192.168.10.20 (192.168.10.20)' can't be established.
ECDSA key fingerprint is SHA256:WMDZ570nPB0kFwRamTzQVSm4YpLzDyrFN8RDGFwsdx4.
ECDSA key fingerprint is MD5:cc:9e:30:9b:b1:25:eb:11:0b:a9:e2:a7:a5:0b:06:03.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.10.20' (ECDSA) to the list of known hosts.
[borg@backup ~]$ exit
logout
Connection to 192.168.10.20 closed.
[root@server .ssh]#
```
</p>
</details>

### Шифрование

Теперь с хоста server (с клиента) инициируем репозиторий с шифрованием (опция --encryption) с именем Backup на хосте backup:

```shell
[root@server ~]# borg init --encryption=repokey-blake2 borg@192.168.10.20:Backup
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "passphrase"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.10.20/./Backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
```

borg попросит ввести passphrase (в моем случае я ввел "passphrase").

Ключ шифрования после инициализации репозитория будет храниться на хосте backup в файле <REPO_DIR>/config:

```shell
[root@backup ~]# cat /home/borg/Backup/config
[repository]
version = 1
segments_per_dir = 1000
max_segment_size = 524288000
append_only = 0
storage_quota = 0
additional_free_space = 0
id = df295a1666f6859a256d723d183fb7d1f87c0cfd129daae9aac124a1806f322f
key = hqlhbGdvcml0aG2mc2hhMjU2pGRhdGHaAZ4vOhu0ejbAR9Es+Lao1wo26QyUnRo140GlXc  
        0Dnw+Bx2GBULJy50mvcoumu+zwQCZUHoKDzIODLQoBqLX6iVXkemHRuezujdzR9MdF65x1
        RrwpCh9CbQc/p1au8ioCku1SXCpPNDOmo1Yoz/cJPEukvxmQZe32Joi6T+u5v/VlyYZCcj
        UjmqrYqTICsaaOIDeqa3JroUikNS/0WeQWxAN3e5aBUNpDBLeBefl2GgxyESRF2wKc6RSN
        Y76YScdRlTk8ZUVWbUAkXecttWIXYujFN1oqzNnVLb0iHhjYxal7q71Ut0nXN2h6e2TASB
        UIYqJbyF4dMs7vLLeQsgZuqyzBIa0rbn9vlrmuIhMLzE5Qoket8DeTAvXYi4P50KIHO5FT
        CbZ0bJ5fYjLexPg/nY/KGbUe7y0YGZMQwYrXkKrSCFXfukrm7lGTj3LE1XHyoYt4u9IebU
        PofwzY7tVSDzksqs6HoU2kBHcU48Z151WcoAbSaNQ6tXwqWbMbUpGeB8DHr+GDGM1DDtaH
        sRfa7+l4dJzkw63GrsXhH+WxzsykaGFzaNoAIKpWS0aXOGzZd0f/CL7U4bTMQYiiv0PhjG
        3YOhttHx51qml0ZXJhdGlvbnPOAAGGoKRzYWx02gAgfTWNxtK7W3x042ON6oxAYCIktFLE
        ErXbLqPStE8L+AundmVyc2lvbgE=
```

При настроенном шифровании passphrase будет запрашиваться каждый раз при запуске процедуры бэкапа. Поэтому для автоматизации бэкапа в скрипте одним из способов является передача passphrase в переменную окружения BORG_PASSPHRASE: export BORG_PASSPHRASE='passphrase'.

### Политика хранения бэкапа

Политика хранения определяется в скрипте командой borg prune - она удаляет из репозитория все архивы, не соответсвующие ни одному из указанных критериев, которые в свою очередь определяются параметрами --keep-*. В данном ДЗ определено хранить бэкапы за последние 3 месяца и по одному за предыдущие 9 месяцев:

```shell
borg prune \
  -v --list \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO} \
  --keep-daily=90 \
  --keep-monthly=9 >> ${LOG}
```

### Автоматическое выполнение бэкапа

По заданию бэкап нужно делать каждые 5 минут. Для этого создадим systemd service и systemd timer:

```shell
[root@server vagrant]# cat /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg /etc backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/root/borg-backup.sh
```

```shell
[root@server vagrant]# cat /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg backup timer

[Timer]
#run hourly
OnBootSec=1min
OnUnitActiveSec=5min
Unit=borg-backup.service

[Install]
WantedBy=multi-user.target
```

Обновим конфигурацию systemd и запустим таймер:

```shell
[root@server vagrant]# systemctl daemon-reload
[root@server vagrant]# systemctl enable --now borg-backup.timer
Created symlink from /etc/systemd/system/multi-user.target.wants/borg-backup.timer to /etc/systemd/system/borg-backup.timer.
```

Выполним задание бэкапа:

```shell
[root@server ~]# bash borg-backup.sh
```

Вывод списка архивов в репозитории Backup:

```shell
[root@server ~]# borg list borg@192.168.10.20:Backup
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/./Backup: 
etc-server-2021-03-10_12:22:41       Wed, 2021-03-10 12:22:42 [d0cfaa86e30d2785a4c5857ed002e1f8d613bd55a0c5c2682c020b21e081ea8e]
```

Видим один единственный архив etc-server-2021-03-10_12:22:41, посмотрим его содержимое:

```shell
[root@server ~]# borg list borg@192.168.10.20:Backup::etc-server-2021-03-10_12:22:41
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/./Backup: 
drwxr-xr-x root   root          0 Wed, 2021-03-10 11:54:06 etc
-rw-r--r-- root   root        450 Wed, 2021-03-10 11:45:00 etc/fstab
-rw------- root   root          0 Thu, 2020-04-30 22:04:55 etc/crypttab
lrwxrwxrwx root   root         17 Thu, 2020-04-30 22:04:55 etc/mtab -> /proc/self/mounts
-rw-r--r-- root   root          7 Wed, 2021-03-10 05:29:30 etc/hostname
-rw-r--r-- root   root       2388 Thu, 2020-04-30 22:08:36 etc/libuser.conf
-rw-r--r-- root   root       2043 Thu, 2020-04-30 22:08:36 etc/login.defs
-rw-r--r-- root   root         37 Thu, 2020-04-30 22:08:36 etc/vconsole.conf
lrwxrwxrwx root   root         25 Thu, 2020-04-30 22:08:36 etc/localtime -> ../usr/share/zoneinfo/UTC
-rw-r--r-- root   root         19 Thu, 2020-04-30 22:08:36 etc/locale.conf
-rw-r--r-- root   root       1186 Thu, 2020-04-30 22:08:37 etc/passwd
---------- root   root        663 Thu, 2020-04-30 22:08:37 etc/shadow
---------- root   root        433 Thu, 2020-04-30 22:08:37 etc/gshadow
-rw-r--r-- root   root        163 Thu, 2020-04-30 22:05:05 etc/.updated
-rw-r--r-- root   root        543 Thu, 2020-04-30 22:08:37 etc/group
drwxr-xr-x root   root          0 Thu, 2020-04-30 22:06:26 etc/X11
drwxr-xr-x root   root          0 Tue, 2020-04-07 14:38:10 etc/X11/xorg.conf.d
drwxr-xr-x root   root          0 Wed, 2018-04-11 04:59:55 etc/X11/applnk
drwxr-xr-x root   root          0 Wed, 2018-04-11 04:59:55 etc/X11/fontpath.d
drwxr-xr-x root   root          0 Wed, 2021-03-10 11:54:02 etc/rpm
-rw-r--r-- root   root         66 Tue, 2020-04-07 22:01:12 etc/rpm/macros.dist
-rw-r--r-- root   root       5134 Wed, 2020-09-30 13:18:48 etc/rpm/macros.perl
-rw-r--r-- root   root         37 Tue, 2020-04-07 22:01:12 etc/centos-release
-rw-r--r-- root   root         51 Tue, 2020-04-07 22:01:12 etc/centos-release-upstream
-rw-r--r-- root   root         23 Tue, 2020-04-07 22:01:12 etc/issue
-rw-r--r-- root   root         22 Tue, 2020-04-07 22:01:12 etc/issue.net
lrwxrwxrwx root   root         21 Thu, 2020-04-30 22:05:05 etc/os-release -> ../usr/lib/os-release
lrwxrwxrwx root   root         14 Thu, 2020-04-30 22:05:05 etc/redhat-release -> centos-release
lrwxrwxrwx root   root         14 Thu, 2020-04-30 22:05:05 etc/system-release -> centos-release
-rw-r--r-- root   root         23 Tue, 2020-04-07 22:01:12 etc/system-release-cpe
-rw-r--r-- root   root       1529 Wed, 2020-04-01 04:29:32 etc/aliases
-rw-r--r-- root   root       2853 Wed, 2020-04-01 04:29:31 etc/bashrc
...
```


