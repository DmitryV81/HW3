Домашнее задание №3 "Работа С LVM"

Выполнение ДЗ я разделил на блоки, отработка комманд представлена в выводе утилиты script

Блок 1 Уменьшить том под / до 8G часть первая

Создаем временный том для / раздела :

pvcreate /dev/sdb

vgcreate vg_root /dev/sdb

lvcreate -n lv_root -l +100%FREE /dev/vg_root

Создаем на нем файловую систему, смонтируем его для переноса данных:

mkfs.xfs /dev/vg_root/lv_root

mount /dev/vg_root/lv_root /mnt

Копируем все данные с / раздела в /mnt/ :

xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

Переконфигурируем загрузчик, чтобы при старте систему перейти в новый / :

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

Обновим образ initrd :

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

Меняем тома в фонкигурационном файле загрузчика :

sed -i 's|rd.lvm.lv=VolGroup00/LogVol00|rd.lvm.lv=vg_root/lv_root|' /boot/grub2/grub.cfg

exit

Перезагружаемся :

shutdown -r 0

Блок 2 Уменьшить том под / до 8G часть вторая

Убеждаемся, что зашли под новым / томом:

lsblk

Удаляем старый раздел :

lvremove /dev/VolGroup00/LogVol00

Создаем новый раздел с размером 8G :

lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

Создаем на нем файловую систему и монтируем его :

mkfs.xfs /dev/VolGroup00/LogVol00

mount /dev/VolGroup00/LogVol00 /mnt

Переносим данные с временного тома / на созданный :

xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

Переконфигурируем загрузчик :

for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done

chroot /mnt/

grub2-mkconfig -o /boot/grub2/grub.cfg

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

Приступаем к выполнению второй части ДЗ : "Выделить том под /var в зеркало".

Создаем зеркало :

pvcreate /dev/sdc /dev/sdd

vgcreate vg_var /dev/sdc /dev/sdd

lvcreate -L 950M -m1 -n lv_var vg_var

Создаем на нем файловую систему и перемещаем туда /var :

mkfs.ext4 /dev/vg_var/lv_var

mount /dev/vg_var/lv_var /mnt

cp -aR /var/* /mnt/      # rsync -avHPSAX /var/ /mnt/

Резервная копия /var :

mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

Монтируем новый var  в каталог /var :

umount /mnt

mount /dev/vg_var/lv_var /var

В файл fstab прописываем монтирование /var :

echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

exit

Перезагружаемся :

shutdown -r 0

Блок 3 "Загружаемся в новый уменьшенный root и выделяем том под /home работаем со снапшотами"

Удаляем временно созданный раздел :

lvremove /dev/vg_root/lv_root

vgremove /dev/vg_root

pvremove /dev/sdb

Выделяем том под /home :

lvcreate -n LogVol_Home -L 2G /dev/VolGroup00

Создаем на нем файловую систему и монтируем его :

mkfs.xfs /dev/VolGroup00/LogVol_Home

mount /dev/VolGroup00/LogVol_Home /mnt/

Переносим содержимое :

cp -aR /home/* /mnt/

Удаляем всё из старого /home :

rm -rf /home/*

Монтируем новый раздел /home :

umount /mnt

mount /dev/VolGroup00/LogVol_Home /home/

Делаем запись в fstab :

echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

Работа со снапшотами :

Создадим 20 файлов :

touch /home/file{1..20}

ll /home/

Сделаем снапшот :

lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

Удалим часть файлов :

rm -f /home/file{11..20}

Восстановим данные из снапшота :

umount /home

lvconvert --merge /dev/VolGroup00/home_snap

mount /home

ll /home/
