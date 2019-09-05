# OTUS ДЗ 7 Загрузка системы (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание

    Работа с загрузчиком
    Цель: Зайти в систему без пароля рута - базовая задача сисадмина ( ну и одно из заданий на любой линуксовой сертификации). Так же нужно уметь управлять поведением загрузчика. Это и будем учиться делать в ДЗ
    1. Попасть в систему без пароля несколькими способами
    2. Установить систему с LVM, после чего переименовать VG
    3. Добавить модуль в initrd

    4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM
    Репозиторий с пропатченым grub: https://yum.rumyantsev.com/centos/7/x86_64/
    PV необходимо инициализировать с параметром --bootloaderareasize 1m
    Критерии оценки: Описать действия, описать разницу между методами получения шелла в процессе загрузки.
    Где получится - используем script, где не получается - словами или копипастой описываем действия.

1. Попасть в систему без пароля:
- Заходим в загрузчик до старта системы (перед тем, как начнется загрузка нужно успеть нажать на клашиву ```e```)
- Затем находим строку ```linux16``` и убираем из нее все значение ```console```, а частности: ```console=tty0 console=ttyS0,115200n8``` и добавляем ```rd.break```, далее нажимаем ```Ctrl+X```
- Произойдет загрузка системы в аварийном режиме, далее выполняем команду перемонтирования корня для чтения и записи - ```mount -o remount,rw /sysroot```, далее ```chroot /sysroot```
- Далее мы можем поменять пароль, выполнив команду ```passwd``` или ```passwd root```
- После смены пароля необходимо создать скрытый файл ```.autorelabel``` в /, выполнив ```touch /.autorelabel```, этот файл нужен, для того чтобы выполнить relabel файлов в системе, если selinux включен и находиться в режиме ```Enforcing```. Без этого вы не сможете залогиниться в систему после ее загрузки. Однако в моем случае автоматически autorelabel не произошел.
- Далее мне пришлось снова зайти в grub до загрузки системы, после чего в строке linux16 передать ядру параметр загрузки ```enforcing=0```, этот параметр говорит ядру, загрузить систему в режиме selinux=Permissive, в этом режиме система безопасности только лишь пишет в лог файл о найденных нарушениях в файлах, но не блокирует их работу.
- Загрузка произошла, теперь мы можем зайти под root, введя измененный пароль
- Далее выполняем команду ```fixfiles -f relabel```, происходит relabeling файлов, selinux перечитывает файлы системы и начинаем им доверять
- После чего заходим в ```/etc/selinux/config``` и приводим строку к ```SELINUX=Enforcing``` или можем выполнить команду ```setenforce 1```, что также включит полностью selinux.
- Далее можем перезагружаться и попробовать войти в систему, это получится сделать.
Как проверить что SELINUX выключен? - Проверить это можно через команду getenforce или увидев, что в файле ```/etc/selinux/config``` SELINUX равен ```Disabled```.

Другой способ:
- Попасть в систему для смены пароля можно также, успев до загрузки зайти в grub, нажам клавишу ```e```, после чего добавить к строке ```linux16``` ```init=/bin/sh``` и убрать из этой же строки ```console=tty0 console=ttyS0,115200n8``` после чего нажать ```Ctrl+X``` для начала загрузки системы
- Далее перемонтируем корень - ```mount -o remount,rw /```, далее меняем пароль командой ```passwd```, далее проверяем включен для SELUX ```cat /etc/selinux/config```, если ```SELINUX=Enforcing```, то меняем или на ```Permissive``` или ```Disabled```, перезагружаемся
- Заходим в систему по новому паролю, если SELINUX включен, то делаем ```fixfilex -f relabel```

2. Установить систему с LVM, после чего переименовать VG:
- Первым делом можем посмотреть какие Volume Group у нас есть командой ```vgs```
- Далее можем поменять название VG vgrename <Название которое есть сейчас> <Новое название>. Можем просмотреть новое название ```ls -l /dev/mapper```
- Далее будем править файлы, чтобы в дальнейшем мы могли загрузиться, первый файл это ```/etc/fstab```, заходим в него и меняем старое название на новое
- Далее заходим в файл ```/etc/default/grub``` и в строке ```GRUB_CMDLINE_LINUX``` изменяем старое название на новое в значениях ```rd.lvm.lv```
- Далее заходим в файл ```/boot/grub2/grub.cfg``` и меняем старые название на новое в строке ```linux16``` нужного меню
- Далее пересоздаем initrd image, в Centos 7 это не обязательно, так как система все равно загружается, однако делаем это чтобы в файлах initrd image поменялся путь монтирования, выполняем ```mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)```.
- Перезагружаем компьютер, теперь у нас система загружается с новым названием VG. Также можно менять названия логических томов.

3. Добавить модуль в initrd:
- Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем 01test, ```mkdir /usr/lib/dracut/modules.d/01test```
- Далее создаем там 2 скрипта - ```module-setup.sh``` и ```test.sh```, делаем их исполняемыми ```chmod +x```. В скрипт ```module-setup.sh``` вписываем:
```
#!/bin/bash

check() { # Функция, которая указывает что модуль должен быть включен по умолчанию
    return 0
}

depends() { # Выводит все зависимости от которых зависит наш модуль
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh" # Запускает скрипт
}
```
В файле ```test.sh```:
```
#!/bin/bash

cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
```
- Далее выполняем команду ```dracut -f -v```
- Перезагружаемся и видим пингвина

4(*). Сконфигурировать систему без отдельного раздела с /boot, а только с LVM:
- ```parted, далее select /dev/sdb, mklabel msdos, mkpart 1 -1```
- ```pvcreate /dev/sdb1/ --bootloaderareasize 1M```
- ```vgcreate newRoot /dev/sdb1```
- ```lvcreate -n root -l 100%FREE newOtus```
Получаем:
```/dev/mapper/newRoot-root```
```
[root@lvm ~]# pvs
  PV         VG      Fmt  Attr PSize   PFree
  /dev/sda1  newRoot lvm2 a--  <10.00g    0
[root@lvm ~]# vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  newRoot   1   1   0 wz--n- <10.00g    0
[root@lvm ~]# lvs
  LV   VG      Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root newRoot -wi-ao---- <10.00g
```
- ```mkfs.ext4 /dev/mapper/newRoot-root```
- ```mkdir /mnt/root```
- ```mount /dev/mapper/newRoot-root /mnt/root```
- ```rsync -avx / /mnt/root; rsync -avx /boot /mnt/root```
- ```mount --rbind /dev/ /mnt/root/dev; mount --rbind /proc /mnt/root/proc; mount --rbind /sys /mnt/root/sys; mount --rbind /run /mnt/root/run```
- ```chroot /mnt/root```
- ```yum-config-manager --add-repo=https://yum.rumyantsev.com/centos/7/x86_64/```
- ```yum install grub2 -y --nogpgcheck```
- ```cat /etc/fstab```, меняем на наш новый lvm```/dev/mapper/newRoot-root /``` и комментируем /boot
- ```nano /etc/default/grub``` и меняем в ```GRUB_CMDLINE_LINUX``` значение rd.lvm.lv на ```rd.lvm.lv=newRoot/root```
- ```grub2-mkconfig -o /boot/grub2/grub.cfg```
- ```dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img```
- ```grub2-install /dev/sdb```
- ```nano /etc/selinux/config``` и выключить значением ```Disabled``` SELINUX
- выходим и загружаемся с нового диска
```
[root@lvm ~]# lsblk
NAME             MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda                8:0    0  10G  0 disk
└─sda1             8:1    0  10G  0 part
  └─newRoot-root 253:0    0  10G  0 lvm  /
```

GRUB:
```
### BEGIN /etc/grub.d/10_linux ###
menuentry 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10
.0-862.2.3.el7.x86_64-advanced-c14b0452-22d6-459f-aff0-c26dfd925547' {
        load_video
        set gfxpayload=keep
        insmod gzio
        insmod part_msdos
        insmod lvm
        insmod ext2
        set root='lvmid/ubAbwR-P8ye-gBeG-KNMG-sdiA-JxqV-kwgcwk/89GBxh-NSIc-dAFo-LXg4-W1jJ-p0bF-rLdNxY'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint='lvmid/ubAbwR-P8ye-gBeG-KNMG-sdiA-JxqV-kwgcwk/89GBxh-NSIc-dAFo-LXg4-W1jJ-p0bF-rLdNxY'  c14b0452-22d6-459f-aff0-c26dfd925547
        else
          search --no-floppy --fs-uuid --set=root c14b0452-22d6-459f-aff0-c26dfd925547
        fi
        linux16 /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/newRoot-root ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=newRoot/root rhgb quiet
        initrd16 /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
}
if [ "x$default" = 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' ]; then default='Advanced options for CentOS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)'; fi;
### END /etc/grub.d/10_linux ###
```
```set root='lvmid/ubAbwR-P8ye-gBeG-KNMG-sdiA-JxqV-kwgcwk/89GBxh-NSIc-dAFo-LXg4-W1jJ-p0bF-rLdNxY'``` - UID VG и LV.
