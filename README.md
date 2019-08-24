# Домашнее задание по занятию №2
OTUS (Linux Administrator) lesson #2

Часть I. Основное задание

1. В предоставленный для ДЗ Vagrantfile добавим 1 диск 

2. Добавим в Vagrantfile скрипт для создания raid массива 10 уровня: 
    - скрипт зануляет суперблоки на дисках;
    - так как есть уверенность в одинаковом размере дисков, собираем raid-массив непосредственно на дисках, не создавая разделы;
    - к raid добавляем hot-spare диск;
    - сохраняем конфигурацию в /etc/mdadm/mdadm.conf
    
После успешного входа в vagrant проверяем собранный raid:

[root@otuslinux vagrant]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE   MOUNTPOINT
sda      8:0    0   40G  0 disk   
└─sda1   8:1    0   40G  0 part   /
sdb      8:16   0  250M  0 disk   
└─md0    9:0    0  496M  0 raid10 
sdc      8:32   0  250M  0 disk   
└─md0    9:0    0  496M  0 raid10 
sdd      8:48   0  250M  0 disk   
└─md0    9:0    0  496M  0 raid10 
sde      8:64   0  250M  0 disk   
└─md0    9:0    0  496M  0 raid10 
sdf      8:80   0  250M  0 disk   
└─md0    9:0    0  496M  0 raid10 


[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdf[4](S) sde[3] sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      
unused devices: <none>

[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
        Raid Level : raid10
<-....->
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       4       8       80        -      spare   /dev/sdf

3. Произведем синтетический вывод из строя одного из дисков в массиве:

[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdf[4] sde[3](F) sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      
unused devices: <none>
[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
<-....->
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       4       8       80        3      active sync set-B   /dev/sdf

       3       8       64        -      faulty   /dev/sde

Диск пометился как faulty, hot-spare занял его место, rebuild прошел успешно.

Произведем вывод еще одного диска:

[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sdf
mdadm: set /dev/sdf faulty in /dev/md0
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sdf[4](F) sde[3](F) sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/3] [UUU_]
      
unused devices: <none>

Массив в состоянии degraded.

Удалим faulty диски из массива:

[root@otuslinux vagrant]# mdadm /dev/md0 --remove /dev/sd{e,f}
mdadm: hot removed /dev/sde from /dev/md0
mdadm: hot removed /dev/sdf from /dev/md0

Предполагая, что диски были заменены добавим обратно в массив

[root@otuslinux vagrant]# mdadm /dev/md0 --add /dev/sd{f,e}
mdadm: added /dev/sdf
mdadm: added /dev/sde
[root@otuslinux vagrant]# cat /proc/mdstat 
Personalities : [raid10] 
md0 : active raid10 sde[5] sdf[4](S) sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      
unused devices: <none>
[root@otuslinux vagrant]# mdadm -D /dev/md0
/dev/md0:
<-....->
    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       1       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       5       8       64        3      active sync set-B   /dev/sde

       4       8       80        -      spare   /dev/sdf

Массив приведен в первоначальное рабочее состояние.

4. Создадим на RAID массиве таблицу разделов GPT и добавим 5 партиций утилитой parted:

[root@otuslinux vagrant]# lsblk                                           
NAME      MAJ:MIN RM  SIZE RO TYPE   MOUNTPOINT
sda         8:0    0   40G  0 disk   
└─sda1      8:1    0   40G  0 part   /
sdb         8:16   0  250M  0 disk   
└─md0       9:0    0  496M  0 raid10 
  ├─md0p1 259:0    0   98M  0 md     
  ├─md0p2 259:1    0   99M  0 md     
  ├─md0p3 259:2    0  100M  0 md     
  ├─md0p4 259:3    0   99M  0 md     
  └─md0p5 259:4    0   98M  0 md     
sdc         8:32   0  250M  0 disk   
└─md0       9:0    0  496M  0 raid10 
  ├─md0p1 259:0    0   98M  0 md     
  ├─md0p2 259:1    0   99M  0 md     
  ├─md0p3 259:2    0  100M  0 md     
  ├─md0p4 259:3    0   99M  0 md     
  └─md0p5 259:4    0   98M  0 md     
sdd         8:48   0  250M  0 disk   
└─md0       9:0    0  496M  0 raid10 
  ├─md0p1 259:0    0   98M  0 md     
  ├─md0p2 259:1    0   99M  0 md     
  ├─md0p3 259:2    0  100M  0 md     
  ├─md0p4 259:3    0   99M  0 md     
  └─md0p5 259:4    0   98M  0 md     
sde         8:64   0  250M  0 disk   
└─md0       9:0    0  496M  0 raid10 
  ├─md0p1 259:0    0   98M  0 md     
  ├─md0p2 259:1    0   99M  0 md     
  ├─md0p3 259:2    0  100M  0 md     
  ├─md0p4 259:3    0   99M  0 md     
  └─md0p5 259:4    0   98M  0 md     
sdf         8:80   0  250M  0 disk   
└─md0       9:0    0  496M  0 raid10 
  ├─md0p1 259:0    0   98M  0 md     
  ├─md0p2 259:1    0   99M  0 md     
  ├─md0p3 259:2    0  100M  0 md     
  ├─md0p4 259:3    0   99M  0 md     
  └─md0p5 259:4    0   98M  0 md  

Создадим на новых партициях файловую систему ext4 и смонтируем в каталоги:

[root@otuslinux vagrant]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G  2.9G   38G   8% /
devtmpfs        488M     0  488M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
tmpfs           100M     0  100M   0% /run/user/1000
/dev/md0p1       91M  1.6M   83M   2% /raid/part1
/dev/md0p2       92M  1.6M   84M   2% /raid/part2
/dev/md0p3       93M  1.6M   85M   2% /raid/part3
/dev/md0p4       92M  1.6M   84M   2% /raid/part4
/dev/md0p5       91M  1.6M   83M   2% /raid/part5

Основное задание выполнено.

Часть II. Задание **

При выполнении второго задания для удобства был создан отдельный Vagrantfile с одним подключенным диском, соответствующим размеру основного боксового диска. Это позволит скопировать таблицу раздела с основного диска "как есть". При необходимости можно создать таблицу разделов вручную.

1. Скопируем таблицу разделов на новый диск: 

sfdisk -d /dev/sda | sfdisk /dev/sdb

2. На новом диске изменим ID раздела с 83 на fd командой toggle из утилиты parted

3. Создадим degraded массив из нового диска и отформатируем:

mdadm --create /dev/md0 --level=1 --raid-devices=2 missing /dev/sdb1
mkfs.ext4 /dev/md0

4. Примонтируем полученный массив в директорию /mnt и скопируем на него текущую систему:

mount /dev/md0 /mnt/ && rsync -axu / /mnt/

5. Примонтируем временные файловые системы и сделаем chroot в новый корень: 

mount --bind /proc /mnt/proc && mount --bind /dev /mnt/dev && mount --bind /sys /mnt/sys && mount --bind /run /mnt/run && chroot /mnt/

6. В новом корне правим /etc/fstab заменяя uuid корневого диска на uuid массива, полученный командой blkid

7. Создаем конфиг mdadm:

mdadm --detail --scan > /etc/mdadm.conf 

8. Пересоздаем initramfs:

dracut /boot/initramfs-$(uname -r).img $(uname -r)

В этом моменте есть проблема, на решение, а точнее поиск которой ушло много времени. По умолчанию dracut собирает initramfs, который не видит raid-массивы и, естественно, с них не грузится.
Решается данная проблема передачей ядру опции rd.auto=1 либо rd.md.uuid= UUID. Для этого укажем опцию в /etc/default/grub в GRUB_CMDLINE_LINUX

GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.auto=1"

9. Обновляем конфигурацию загрузчика и устанавливаем его на диск:

grub2-mkconfig -o /boot/grub2/grub.cfg && grub2-install /dev/sdb

10. Перезагрузимся и через консоль virtualbox выберем загрузочным второй диск:

[root@otuslinux vagrant]# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda       8:0    0   40G  0 disk  
└─sda1    8:1    0   40G  0 part  
sdb       8:16   0 48.8G  0 disk  
└─sdb1    8:17   0   40G  0 part  
  └─md0   9:0    0   40G  0 raid1 /

Система перенесена успешно. 
После переноса SElinux не пускал через vagrant ssh, было решено отключением SElinux.

11. Для использования первого диска в качестве участника массива, затоглим его с помощью parted в raid и добавим к текущему массиву, записав загрузчик:

mdadm --manage /dev/md0 --add /dev/sda1
grub2-install /dev/sda

Дождемся, когда массив пересоберется:
 
watch cat /proc/mdadm 

Every 2.0s: cat /proc/mdstat      

Personalities : [raid1]
md0 : active raid1 sda1[2] sdb1[1]
      41908224 blocks super 1.2 [2/1] [_U]
      [==============>......]  recovery = 74.0% (31035136/41908224) finish=0.9min speed=200006K/sec

unused devices: <none>

Теперь можно загружаться с первого диска.

[vagrant@otuslinux ~]$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda       8:0    0   40G  0 disk  
└─sda1    8:1    0   40G  0 part  
  └─md0   9:0    0   40G  0 raid1 /
sdb       8:16   0 48.8G  0 disk  
└─sdb1    8:17   0   40G  0 part  
  └─md0   9:0    0   40G  0 raid1 /

Массив успешно работает. Задание выполнено.
