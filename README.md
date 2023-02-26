# less-5 
     ZFS   

------Отрабатываем навыки работы с созданием томов export/import и установкой параметров------  
Все действия произведены от пользователя root. sudo -i 

------ определить алгоритм с наилучшим сжатием--------  
Смотрим список всех дисков, которые есть в виртуальной машине  
 root@zfs ~]# lsblk  
 NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
 sda      8:0    0   40G  0 disk   
 `-sda1   8:1    0   40G  0 part /  
 sdb      8:16   0  512M  0 disk   
 sdc      8:32   0  512M  0 disk   
 sdd      8:48   0  512M  0 disk   
 sde      8:64   0  512M  0 disk   
 sdf      8:80   0  512M  0 disk   
 sdg      8:96   0  512M  0 disk     
 sdh      8:112  0  512M  0 disk   
 sdi      8:128  0  512M  0 disk   
Создаём пул из двух дисков в режиме RAID 1  
 [root@zfs ~]# zpool create space1 mirror /dev/sdb /dev/sdc  
 [root@zfs ~]# zpool create space2 mirror /dev/sdd /dev/sde  
 [root@zfs ~]# zpool create space3 mirror /dev/sdf /dev/sdg  
 [root@zfs ~]# zpool create space4 mirror /dev/sdh /dev/sdi  
Смотрим информацию о пулах  
 [root@zfs ~]# zpool list  
 NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT  
 space1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
 space2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
 space3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
 space4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
Добавим разные алгоритмы сжатия в каждую файловую систему   
 [root@zfs ~]# zfs set compression=lzjb space1  
 [root@zfs ~]# zfs set compression=lz4 space2  
 [root@zfs ~]# zfs set compression=gzip-9 space3  
 [root@zfs ~]# zfs set compression=zle space4  
Проверим, что все файловые системы имеют разные методы сжатия
 [root@zfs ~]# zfs get all | grep compression  
 space1  compression           lzjb                   local  
 space2  compression           lz4                    local  
 space3  compression           gzip-9                 local  
 space4  compression           zle                    local  
Скачаем один и тот же текстовый файл во все пулы  
 [root@zfs ~]# for i in {1..4}; do wget -P /space$i https://www.gutenberg.org/ebooks/2600.txt.utf-8; done  
   --2023-02-26 19:50:57--  https://www.gutenberg.org/ebooks/2600.txt.utf-8  
   Resolving www.gutenberg.org (www.gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47  
   Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:443... connected.  
   HTTP request sent, awaiting response... 302 Found  
   Location: https://www.gutenberg.org/cache/epub/2600/pg2600.txt [following]  
   --2023-02-26 19:51:09--  https://www.gutenberg.org/cache/epub/2600/pg2600.txt  
   Reusing existing connection to www.gutenberg.org:443.  
   HTTP request sent, awaiting response... 200 OK  
   Length: 3359372 (3.2M) [text/plain]  
   Saving to: '/space1/2600.txt.utf-8.1'  
  100%[======================================================================================================================================>] 3,359,372    905KB/s   in 3.6s   

  2023-02-26 19:51:12 (905 KB/s) - '/space1/2600.txt.utf-8.1' saved [3359372/3359372]  

  100%[======================================================================================================================================>] 3,359,372    627KB/s   in 5.5s   
 
  2023-02-26 19:51:37 (593 KB/s) - '/space2/2600.txt.utf-8.1' saved [3359372/3359372]  

  100%[======================================================================================================================================>] 3,359,372    671KB/s   in 9.3s   

  2023-02-26 19:51:59 (352 KB/s) - '/space3/2600.txt.utf-8.1' saved [3359372/3359372]  

  100%[======================================================================================================================================>] 3,359,372    636KB/s   in 9.2s   

  2023-02-26 19:52:23 (357 KB/s) - '/space4/2600.txt.utf-8' saved [3359372/3359372]  
Проверим, что файл был скачан во все пулы  
 [root@zfs ~]# ls -l /space*  
 /space1:  
 total 4885  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8.1  

 /space2:  
 total 4081  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8.1  

 /space3:  
 total 2477  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8.1  

 /space4:  
 total 3287  
 -rw-r--r--. 1 root root 3359372 Feb  2 09:16 2600.txt.utf-8  
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов  
 root@zfs ~]# zfs list  
 NAME     USED  AVAIL     REFER  MOUNTPOINT  
 space1  4.86M   347M     4.79M  /space1  
 space2  4.08M   348M     4.01M  /space2  
 space3  2.51M   349M     2.44M  /space3  
 space4  3.30M   349M     3.23M  /space4  
 [root@zfs ~]# zfs get all | grep compressratio | grep -v ref  
 space1  compressratio         1.36x                  -  
 space2  compressratio         1.62x                  -  
 space3  compressratio         2.66x                  -  
 space4  compressratio         1.01x                 
Алгоритм gzip-9 самый эффективный по сжатию 

-----Определить настройки pool’a------  
Скачиваем архив в домашний каталог  
 [root@zfs ~]# wget -O archive.tar.gz --no-check-certificate "https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download"  
 --2023-02-26 20:09:10--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download  
 Resolving drive.google.com (drive.google.com)... 173.194.222.194, 2a00:1450:4010:c02::c2  
 Connecting to drive.google.com (drive.google.com)|173.194.222.194|:443... connected.  
 HTTP request sent, awaiting response... 302 Found  
 Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]  
 --2023-02-26 20:09:10--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download  
 Reusing existing connection to drive.google.com:443.  
 HTTP request sent, awaiting response... 303 See Other  
 Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/dufek6e1dltdhtkd7iustdgvo36lb4a2/1677442125000/16189157874053420687/*/  1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=c4728d8c-f3f0-4a42-81bc-92065bfd80c9 [following]
 Warning: wildcards not supported in HTTP.  
 --2023-02-26 20:09:14--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/dufek6e1dltdhtkd7iustdgvo36lb4a2/1677442125000/  16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=c4728d8c-f3f0-4a42-81bc-92065bfd80c9  
 Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 64.233.161.132, 2a00:1450:4010:c06::84  
 Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|64.233.161.132|:443... connected.  
 HTTP request sent, awaiting response... 200 OK  
 Length: 7275140 (6.9M) [application/x-gzip]  
 Saving to: 'archive.tar.gz'  

 100%[=======================================================================================================================================>] 7,275,140   10.0MB/s   in 0.7s   
 2023-02-26 20:09:15 (10.0 MB/s) - 'archive.tar.gz' saved [7275140/7275140]  
Разархивируем его   
 root@zfs ~]# tar -xzvf archive.tar.gz  
 zpoolexport/  
 zpoolexport/filea  
 zpoolexport/fileb  
Проверим, возможно ли импортировать данный каталог в пул  
 [root@zfs ~]# zpool import -d zpoolexport/  
   pool: otus  
     id: 6554193320433390805  
  state: ONLINE  
 action: The pool can be imported using its name or numeric identifier.  
 config:  

	otus                         ONLINE  
	  mirror-0                   ONLINE  
	    /root/zpoolexport/filea  ONLINE  
	    /root/zpoolexport/fileb  ONLINE  
Сделаем импорт данного пула к нам в ОС
 [root@zfs ~]# zpool import -d zpoolexport/ otus   
Информация о составе импортированного пула
 [root@zfs ~]# zpool status  
  pool: otus  
 state: ONLINE  
  scan: none requested  
 config:  

	NAME                         STATE     READ WRITE CKSUM  
	otus                         ONLINE       0     0     0  
	  mirror-0                   ONLINE       0     0     0  
	    /root/zpoolexport/filea  ONLINE       0     0     0  
	    /root/zpoolexport/fileb  ONLINE       0     0     0  

 errors: No known data errors 

  pool: space1  
 state: ONLINE  
  scan: none requested  
 config:  

	NAME        STATE     READ WRITE CKSUM  
	space1      ONLINE       0     0     0  
	  mirror-0  ONLINE       0     0     0  
	    sdb     ONLINE       0     0     0  
	    sdc     ONLINE       0     0     0  

 errors: No known data errors  и т.д.
Определим настройки пула  
 [root@zfs ~]# zpool get all otus  
 NAME  PROPERTY                       VALUE                          SOURCE  
 otus  size                           480M                           -  
 otus  capacity                       0%                             -
 otus  altroot                        -                              default  
 otus  health                         ONLINE                         -  
 otus  guid                           6554193320433390805            -  
 otus  version                        -                              default  
 otus  bootfs                         -                              default  
 otus  delegation                     on                             default  
 otus  autoreplace                    off                            default  
 otus  cachefile                      -                              default  
 otus  failmode                       wait                           default  
 otus  listsnapshots                  off                            default  
 otus  autoexpand                     off                            default  
 otus  dedupditto                     0                              default  
 otus  dedupratio                     1.00x                          -  
 otus  free                           478M                           -  
 otus  allocated                      2.09M                          -  
 otus  readonly                       off                            -  
 otus  ashift                         0                              default  
 otus  comment                        -                              default  
 otus  expandsize                     -                              -  
 otus  freeing                        0                              -  
 otus  fragmentation                  0%                             -  
 otus  leaked                         0                              -  
 otus  multihost                      off                            default  
 otus  checkpoint                     -                              -  
 otus  load_guid                      4819895864187526099            -  
 otus  autotrim                       off                            default  
 otus  feature@async_destroy          enabled                        local  
 otus  feature@empty_bpobj            active                         local  
 otus  feature@lz4_compress           active                         local  
 otus  feature@multi_vdev_crash_dump  enabled                        local  
 otus  feature@spacemap_histogram     active                         local  
 otus  feature@enabled_txg            active                         local  
 otus  feature@hole_birth             active                         local  
 otus  feature@extensible_dataset     active                         local  
 otus  feature@embedded_data          active                         local  
 otus  feature@bookmarks              enabled                        local  
 otus  feature@filesystem_limits      enabled                        local  
 otus  feature@large_blocks           enabled                        local  
 otus  feature@large_dnode            enabled                        local  
 otus  feature@sha512                 enabled                        local  
 otus  feature@skein                  enabled                        local   
 otus  feature@edonr                  enabled                        local  
 otus  feature@userobj_accounting     active                         local  
 otus  feature@encryption             enabled                        local  
 otus  feature@project_quota          active                         local  
 otus  feature@device_removal         enabled                        local  
 otus  feature@obsolete_counts        enabled                        local  
 otus  feature@zpool_checkpoint       enabled                        local  
 otus  feature@spacemap_v2            active                         local  
 otus  feature@allocation_classes     enabled                        local  
 otus  feature@resilver_defer         enabled                        local  
 otus  feature@bookmark_v2            enabled                        local  
Размер  
 [root@zfs ~]# zfs get available otus  
 NAME  PROPERTY   VALUE  SOURCE  
 otus  available  350M   -  
Тип  
 [root@zfs ~]# zfs get readonly otus 
 NAME  PROPERTY  VALUE   SOURCE 
 otus  readonly  off     default 
Значение recordsize:  
 [root@zfs ~]# zfs get recordsize otus  
 NAME  PROPERTY    VALUE    SOURCE  
 otus  recordsize  128K     local  
Тип сжатия  
 [root@zfs ~]# zfs get compression otus  
 NAME  PROPERTY     VALUE     SOURCE  
 otus  compression  zle       local  
Тип контрольной суммы   
 [root@zfs ~]# zfs get checksum otus  
 NAME  PROPERTY  VALUE      SOURCE  
 otus  checksum  sha256     local  

------Работа со снапшотом, поиск сообщения от преподавателя-----   
Скачаем файл  
 [root@zfs ~]# wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&e"  
 --2023-02-26 20:35:46--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&e  
 Resolving drive.google.com (drive.google.com)... 64.233.165.194, 2a00:1450:4010:c1c::c2  
 Connecting to drive.google.com (drive.google.com)|64.233.165.194|:443... connected.  
 HTTP request sent, awaiting response... 302 Found  
 Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&e [following]  
 --2023-02-26 20:35:46--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&e  
 Reusing existing connection to drive.google.com:443.  
 HTTP request sent, awaiting response... 303 See Other  
 Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/3b574c7m77btbbgjavke5ona3e6sbht0/1677443700000/16189157874053420687/*/   1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?uuid=ad8651ad-1b0c-4d8a-842c-1545d37fb8ac [following]
 Warning: wildcards not supported in HTTP.
 --2023-02-26 20:35:50--  https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/3b574c7m77btbbgjavke5ona3e6sbht0/1677443700000/  16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?uuid=ad8651ad-1b0c-4d8a-842c-1545d37fb8ac  
 Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 64.233.163.132, 2a00:1450:4010:c0b::84  
 Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|64.233.163.132|:443... connected.  
 HTTP request sent, awaiting response... 200 OK  
 Length: 5432736 (5.2M) [application/octet-stream]  
 Saving to: 'otus_task2.file'  

 100%[=================================================================================================================================>] 5,432,736   9.99MB/s   in 0.5s     

 2023-02-26 20:35:51 (9.99 MB/s) - 'otus_task2.file' saved [5432736/5432736]  

Восстановим файловую систему из снапшота  
 [root@zfs ~]# zfs receive otus/test@today <otus_task2.file  
Найдем в каталоге файл с именем secret_message  
 [root@zfs ~]# find /otus/test -name "secret_message"  
 /otus/test/task1/file_mess/secret_message  
Посмотрим содержимое найденного файла  
 [root@zfs ~]# cat /otus/test/task1/file_mess/secret_message  
 https://github.com/sindresorhus/awesome  - это ссылка на Github.


