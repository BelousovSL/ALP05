# Название выполняемого задания;
Научится создавать и работать с логическими томами;

# Текст задания
На имеющемся образе bento/ubuntu-24.04

Уменьшить том под / до 8G.
Выделить том под /home.
Выделить том под /var - сделать в mirror.
/home - сделать том для снапшотов.
Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор).
Работа со снапшотами:
сгенерить файлы в /home/;
снять снапшот;
удалить часть файлов;
восстановится со снапшота.

# Инструкция

vagrant up

## Уменьшить том под / до 8G.

### Мысли в слух.
На сколько я понял под ext4 нельзя уменьшить файловую систему без демонтирования, а root мы не можем демонтировтаь. Поэтому воспользуемся LiveCD.


Для начала проверим что у нас с монтированием. Подключаемся к ВМ.

```bash
vagrant@vagrant:~$ df -h

Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                               97M  972K   96M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  3.3G   26G  12% /
tmpfs                              481M     0  481M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G   96M  1.7G   6% /boot
tmpfs                               97M   12K   97M   1% /run/user/1000
```

Выключаем эту машину и создаем новую. 

```code
vagrant halt
```

Я использую гипервизор VirtualBox. На основе 'ubuntu-24.04-desktop-amd64.iso'. Дополнительно в настройках подключаю диск созданный на этапе выше.

Запускаю новую ВМ...  Пропуская языки и все прочее... Выбираю "Try Ubuntu"

В терминале
```code
sudo lvreduce -L 8G /dev/mapper/ubuntu--vg-ubuntu--lv -r
```

Выключаем ВМ, и запускаем созданную на основе Vagrant

Проверяем результат
```bash
vagrant@vagrant:~$ df -h

Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                               97M  964K   96M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  7.8G  3.3G  4.1G  45% /
tmpfs                              481M     0  481M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G   96M  1.7G   6% /boot
tmpfs                               97M   12K   97M   1% /run/user/1000

```

Вывод Root стал 8GB

## Выделить том под /home. Монтирование в fstab

Создаем логический том на 1GB.
```bash
sudo lvcreate -L 1G -n home ubuntu-vg
```

Форматиуем его в Ext4
```bash
sudo mkfs.ext4 /dev/mapper/ubuntu--vg-home
```
Создаем временный каталог
```bash
sudo mkdir /homenew
```

Монтируем к нему нашу новый логический том
```bash
sudo mount /dev/mapper/ubuntu--vg-home /homenew/
```

Переносим текущее содержимое каталога '/home'
```bash
sudo mv /home/* /homenew/
```

Отмантируем 
```bash
sudo umount /dev/mapper/ubuntu--vg-home
```

В файле '/etc/fstab' добавляем строку
```text
/dev/mapper/ubuntu--vg-home /home ext4 defaults 0 1
```
Монтируем

```bash
sudo systemctl daemon-reload
sudo mount -a
```

Проверяем 
```bash
vagrant@vagrant:~$ df -h

Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                               97M  976K   96M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  7.8G  3.3G  4.1G  45% /
tmpfs                              481M     0  481M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G   96M  1.7G   6% /boot
tmpfs                               97M   12K   97M   1% /run/user/1000
/dev/mapper/ubuntu--vg-home        974M   56K  907M   1% /home
```

```bash
vagrant@vagrant:~$ ll /home/vagrant/

total 32
drwxr-x--- 4 vagrant vagrant 4096 Feb  2 16:50 ./
drwxr-xr-x 4 root    root    4096 Feb  2 16:52 ../
-rw-r--r-- 1 vagrant vagrant  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 vagrant vagrant 3771 Mar 31  2024 .bashrc
drwx------ 2 vagrant vagrant 4096 Feb  2 16:50 .cache/
-rw-r--r-- 1 vagrant vagrant  807 Mar 31  2024 .profile
drwx------ 2 vagrant vagrant 4096 Feb  2 16:50 .ssh/
-rw-r--r-- 1 vagrant vagrant    0 Apr 29  2024 .sudo_as_admin_successful
-rw-r--r-- 1 vagrant vagrant    6 Apr 29  2024 .vbox_version
```

## Выделить том под /var - сделать в mirror. Монтирование в fstab
 
 Добавляем новое блочное устройство
 ```bash
 sudo pvcreate /dev/sdb
 ```

 Вновь созданный pv добавляем в lv
 ```bash
 sudo vgextend ubuntu-vg /dev/sdb
 ```

Создаем зеркальный логический том
```bash
sudo lvcreate -L 500MB -m1 -n var ubuntu-vg
``` 

Смотрим результат
```bash
vagrant@vagrant:~$ sudo lvs

  LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home      ubuntu-vg -wi-ao----   1.00g                                                    
  ubuntu-lv ubuntu-vg -wi-ao----   8.00g                                                    
  var       ubuntu-vg rwi-a-r--- 500.00m                                    100.00 
```

Далее как с home
```bash
sudo mkfs.ext4 /dev/mapper/ubuntu--vg-var
sudo mkdir /varnew
sudo mount /dev/mapper/ubuntu--vg-var /varnew/
sudo mv /var/* /varnew/
sudo umount /dev/mapper/ubuntu--vg-var
В fstab /dev/mapper/ubuntu--vg-var /var ext4 defaults 0 1
sudo systemctl daemon-reload
sudo mount -a
```

Проверяем результат
```bash
vagrant@belousovRaid:/home$ sudo lvs

  LV        VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home      ubuntu-vg -wi-ao----   1.00g                                                    
  ubuntu-lv ubuntu-vg -wi-ao----   8.00g                                                    
  var       ubuntu-vg rwi-aor--- 500.00m                                    100.00            
```

```bash
ll /var

total 68
drwxr-xr-x 14 root root    4096 Feb  2 17:12 ./
drwxr-xr-x 25 root root    4096 Feb  2 17:09 ../
drwxr-xr-x  2 root root    4096 Feb  2 17:07 backups/
drwxr-xr-x 14 root root    4096 Apr 29  2024 cache/
drwxrwxrwt  2 root root    4096 Apr 23  2024 crash/
drwxr-xr-x 41 root root    4096 Apr 29  2024 lib/
drwxrwsr-x  2 root staff   4096 Apr 22  2024 local/
lrwxrwxrwx  1 root root       9 Apr 23  2024 lock -> /run/lock/
drwxrwxr-x  8 root syslog  4096 Apr 29  2024 log/
drwx------  2 root root   16384 Feb  2 17:09 lost+found/
drwxrwsr-x  2 root mail    4096 Apr 23  2024 mail/
drwxr-xr-x  2 root root    4096 Apr 23  2024 opt/
lrwxrwxrwx  1 root root       4 Apr 23  2024 run -> /run/
drwxr-xr-x  2 root root    4096 Mar 31  2024 snap/
drwxr-xr-x  4 root root    4096 Apr 23  2024 spool/
drwxrwxrwt  6 root root    4096 Feb  2 17:07 tmp/

```

## Выделить том под /var - сделать в mirror.
/home - сделать том для снапшотов.
Работа со снапшотами:
сгенерить файлы в /home/;
снять снапшот;
удалить часть файлов;
восстановится со снапшота.

Создаем в домашней директории файл

```bash
sudo echo 'other data' >  ~/text
```

Проверяем 
```bash
vagrant@belousovRaid:/home$ ll ~/

total 40
drwxr-x--- 5 vagrant vagrant 4096 Feb  2 17:22 ./
drwxr-xr-x 4 root    root    4096 Feb  2 17:20 ../
-rw-r--r-- 1 vagrant vagrant  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 vagrant vagrant 3771 Mar 31  2024 .bashrc
drwx------ 2 vagrant vagrant 4096 Apr 29  2024 .cache/
drwxrwxr-x 3 vagrant vagrant 4096 Feb  2 17:12 .local/
-rw-r--r-- 1 vagrant vagrant  807 Mar 31  2024 .profile
drwx------ 2 vagrant vagrant 4096 Feb  2 17:07 .ssh/
-rw-r--r-- 1 vagrant vagrant    0 Apr 29  2024 .sudo_as_admin_successful
-rw-rw-r-- 1 vagrant vagrant   11 Feb  2 17:22 text
-rw-r--r-- 1 vagrant vagrant    6 Apr 29  2024 .vbox_version

```

Создаем снимок
```bash
sudo lvcreate -L 1G -s -n home-snap /dev/mapper/ubuntu--vg-home
```

Удаляем файл
```bash
rm ~/text
```

Проверяем 
```bash
vagrant@belousovRaid:/home$ ll ~/

total 36
drwxr-x--- 5 vagrant vagrant 4096 Feb  2 17:24 ./
drwxr-xr-x 4 root    root    4096 Feb  2 17:20 ../
-rw-r--r-- 1 vagrant vagrant  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 vagrant vagrant 3771 Mar 31  2024 .bashrc
drwx------ 2 vagrant vagrant 4096 Apr 29  2024 .cache/
drwxrwxr-x 3 vagrant vagrant 4096 Feb  2 17:12 .local/
-rw-r--r-- 1 vagrant vagrant  807 Mar 31  2024 .profile
drwx------ 2 vagrant vagrant 4096 Feb  2 17:07 .ssh/
-rw-r--r-- 1 vagrant vagrant    0 Apr 29  2024 .sudo_as_admin_successful
-rw-r--r-- 1 vagrant vagrant    6 Apr 29  2024 .vbox_version
```

Откатываемся
```bash
sudo lvconvert --merge /dev/mapper/ubuntu--vg-home--snap
```

Проверяем

```bash
vagrant@belousovRaid:/home$ ll ~/

total 40
drwxr-x--- 5 vagrant vagrant 4096 Feb  2 17:22 ./
drwxr-xr-x 4 root    root    4096 Feb  2 17:20 ../
-rw-r--r-- 1 vagrant vagrant  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 vagrant vagrant 3771 Mar 31  2024 .bashrc
drwx------ 2 vagrant vagrant 4096 Apr 29  2024 .cache/
drwxrwxr-x 3 vagrant vagrant 4096 Feb  2 17:12 .local/
-rw-r--r-- 1 vagrant vagrant  807 Mar 31  2024 .profile
drwx------ 2 vagrant vagrant 4096 Feb  2 17:07 .ssh/
-rw-r--r-- 1 vagrant vagrant    0 Apr 29  2024 .sudo_as_admin_successful
-rw-rw-r-- 1 vagrant vagrant   11 Feb  2 17:22 text
-rw-r--r-- 1 vagrant vagrant    6 Apr 29  2024 .vbox_version
```