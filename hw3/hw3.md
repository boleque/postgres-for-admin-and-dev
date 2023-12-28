# Домашнее задание
## Установка и настройка PostgreSQL

1. создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
```
Создан ec2 инстанс в aws
```
2. поставьте на нее PostgreSQL 15 через sudo apt
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
3. проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```
ubuntu@ip-172-31-25-37:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
4. зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым

```
sudo -u postgres psql
```

```
-- cоздаем новую БД
CREATE DATABASE otus;

--- Подключаемся
\c otus

-- Создаем и наполняем таблицу persons
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov');

-- Проверяем что данные на месте:
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

5. остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop\
```
sudo systemctl stop postgresql@15-main
```
6. создайте новый диск к ВМ размером 10GB\
```
Создан aws volume размером 10GB
```
7. добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk\
```
Проинициализирован диск согласно инструкции https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html
-- получаем информацию обо всех устройствах
root@ip-172-31-25-37:/# sudo lsblk -f
NAME         FSTYPE FSVER LABEL           UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                                0   100% /snap/amazon-ssm-agent/7628
loop1                                                                                0   100% /snap/core18/2812
loop2                                                                                0   100% /snap/core20/2015
loop3                                                                                0   100% /snap/lxd/24322
loop4                                                                                0   100% /snap/snapd/20290
nvme1n1                                                                                       
nvme0n1                                                                                       
├─nvme0n1p1  ext4   1.0   cloudimg-rootfs 9e71e708-e903-4c26-8506-d85b84605ba0    5.6G    26% /
├─nvme0n1p14                                                                                  
└─nvme0n1p15 vfat   FAT32 UEFI            A62D-E731                              98.3M     6% /boot/efi

-- проверяем что файловая система отсутствует на устройстве nvme1n1
root@ip-172-31-25-37:/# sudo file -s /dev/nvme1n1
/dev/nvme1n1: data

root@ip-172-31-25-37:/# sudo mkfs -t ext4 /dev/nvme1n1 
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: 8f46d9ab-13f4-4936-b292-50185391f10c
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
-- проверяем что файловая система создана 
root@ip-172-31-25-37:/# sudo file -s /dev/nvme1n1
/dev/nvme1n1: Linux rev 1.0 ext4 filesystem data, UUID=8f46d9ab-13f4-4936-b292-50185391f10c (extents) (64bit) (large files) (huge files)

-- создаем директорию mnt/data и монтируем туда устройство 
root@ip-172-31-25-37:/# mkdir mnt/data
root@ip-172-31-25-37:/# mount /dev/nvme1n1 /mnt/data

-- проверяем результат
root@ip-172-31-25-37:/# df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/root        7.6G  2.0G  5.6G  27% /
tmpfs            1.9G  1.1M  1.9G   1% /dev/shm
tmpfs            770M  844K  769M   1% /run
tmpfs            5.0M     0  5.0M   0% /run/lock
/dev/nvme0n1p15  105M  6.1M   99M   6% /boot/efi
tmpfs            385M  4.0K  385M   1% /run/user/1000
/dev/nvme1n1     9.8G   24K  9.3G   1% /mnt/data
```
8. перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)\
```
ubuntu@ip-172-31-25-37:~$ df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/root        7.6G  2.0G  5.6G  27% /
tmpfs            1.9G  1.1M  1.9G   1% /dev/shm
tmpfs            770M  844K  769M   1% /run
tmpfs            5.0M     0  5.0M   0% /run/lock
/dev/nvme1n1p15  105M  6.1M   99M   6% /boot/efi
tmpfs            385M  4.0K  385M   1% /run/user/1000
```
9. сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```
/dev/sdf/data - chown -R postgres:postgres /dev/sdf/data
ubuntu@ip-172-31-25-37:/mnt$ ll
total 12
drwxr-xr-x  3 root     root     4096 Dec 28 12:47 ./
drwxr-xr-x 19 root     root     4096 Dec 28 12:51 ../
drwxr-xr-x  2 postgres postgres 4096 Dec 28 12:47 data/
```
10. перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
```
ubuntu@ip-172-31-25-37:/mnt$ sudo mv /var/lib/postgresql/15 /mnt/data
ubuntu@ip-172-31-25-37:/mnt$ ll /mnt/data/
total 12
drwxr-xr-x 3 postgres postgres 4096 Dec 28 12:59 ./
drwxr-xr-x 3 root     root     4096 Dec 28 12:47 ../
drwxr-xr-x 3 postgres postgres 4096 Dec 28 09:36 15/
```
11. попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```
ubuntu@ip-172-31-25-37:/mnt$ sudo systemctl start postgresql@15-main
Job for postgresql@15-main.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@15-main.service" and "journalctl -xeu postgresql@15-main.service" for details.
```
12. напишите получилось или нет и почему
```
Не удалось запустить кластер, т.к. отсутствует указанная в конфигурации data_directory
░░ A start job for unit postgresql@15-main.service has begun execution.
░░ 
░░ The job identifier is 891.
Dec 28 13:05:17 ip-172-31-25-37 postgresql@15-main[961]: Error: /var/lib/postgresql/15/main is not accessible or does not exist
Dec 28 13:05:17 ip-172-31-25-37 systemd[1]: postgresql@15-main.service: Can't open PID file /run/postgresql/15-main.pid (yet?) after start: Operation not permitted
Dec 28 13:05:17 ip-172-31-25-37 systemd[1]: postgresql@15-main.service: Failed with result 'protocol'.
░░ Subject: Unit failed
```
13. задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его\
14. напишите что и почему поменяли
```
Поменял аттрибут data_directory в конфигурационном файле /etc/postgresql/15/main/postgresql.conf
старое значение: data_directory = '/var/lib/postgresql/15/main'
новое значение: data_directory = '/mnt/data/15/main'
```
15. попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start\
16. напишите получилось или нет и почему
```
Кластер запустился
root@ip-172-31-25-37:/mnt# sudo systemctl start postgresql@15-main
root@ip-172-31-25-37:/mnt# sudo systemctl status postgresql@15-main
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Thu 2023-12-28 13:23:39 UTC; 4s ago
    Process: 1032 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
   Main PID: 1037 (postgres)
      Tasks: 6 (limit: 4598)
     Memory: 18.6M
        CPU: 150ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─1037 /usr/lib/postgresql/15/bin/postgres -D /mnt/data/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─1038 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─1039 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─1041 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─1042 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             └─1043 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >

Dec 28 13:23:37 ip-172-31-25-37 systemd[1]: Starting PostgreSQL Cluster 15-main...
Dec 28 13:23:39 ip-172-31-25-37 systemd[1]: Started PostgreSQL Cluster 15-main.
```

17. зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
Данные на месте
root@ip-172-31-25-37:/mnt# sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```