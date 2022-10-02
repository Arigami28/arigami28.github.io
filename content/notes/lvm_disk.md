---
author: "Roman Andreev"
title: "LVM commands"
date: "2022-09-14"
description: "Sample lvm command."
tags: ["lvm","linux"]
categories: ["linux"]
series: ["Linux Guide"]
aliases: ["lvm_commands"]
ShowToc: true
TocOpen: true

---
Тут приведены команды для работы с LVM в linux в 90% случаев их хватает с головой, чтобы создать/растянуть раздел в системе.
<!--more-->

Этой командой можно посмотреть физически воткнутые диски в ваш сервер:

    на Debian
    lsblk -fap
    
    NAME                                            ALIGNMENT MIN-IO OPT-IO PHY-SEC LOG-SEC ROTA SCHED    RQ-SIZE
    sr0                                                     0    512      0     512     512    1 deadline     128
    sda                                                     0 262144 262144     512     512    1 deadline     128
    ├─sda2                                                  0 262144 262144     512     512    1 deadline     128
    │ ├─vghdd-lvRoot (dm-1)                                 0 262144 262144     512     512    1              128
    │ ├─vghdd-nyc--1--palandov--test--ubuntu (dm-6)         0 262144 262144     512     512    1              128
    │ ├─vghdd-lvVarLog (dm-4)                               0 262144 262144     512     512    1              128
    │ ├─vghdd-lvTmp (dm-2)                                  0 262144 262144     512     512    1              128
    │ ├─vghdd-lvHome (dm-0)                                 0 262144 262144     512     512    1              128
    │ ├─vghdd-seriesdb (dm-5)                               0 262144 262144     512     512    1              128
    │ └─vghdd-lvVarLib (dm-3)                               0 262144 262144     512     512    1              128
    └─sda1                                                  0 262144 262144     512     512    1 deadline     128
    
    на Ubuntu
    lsblk
    
    NAME                                            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    sr0                                              11:0    1  1024M  0 rom
    sda                                               8:0    0 931.5G  0 disk
    ├─sda2                                            8:2    0 930.6G  0 part
    │ ├─vghdd-lvRoot (dm-1)                         253:1    0    80G  0 lvm  /
    │ ├─vghdd-nyc--1--palandov--test--ubuntu (dm-6) 253:6    0    30G  0 lvm
    │ ├─vghdd-lvVarLog (dm-4)                       253:4    0   100G  0 lvm  /var/log
    │ ├─vghdd-lvTmp (dm-2)                          253:2    0   1.9G  0 lvm  /tmp
    │ ├─vghdd-lvHome (dm-0)                         253:0    0   9.3G  0 lvm  /home
    │ ├─vghdd-seriesdb (dm-5)                       253:5    0   300G  0 lvm  /opt/seriesdb
    │ └─vghdd-lvVarLib (dm-3)                       253:3    0    40G  0 lvm  /var/lib
    └─sda1                                            8:1    0 953.5M  0 part /boot

Структура LVM состоит из трех слоев:

* Физический том (один или несколько), Physical Volume (PV)
* Группа физических томов, Volume Group (VG)
* Логический том, который и будет доступен программам, Logical Volume (LV)

Краткая информация по слоям:

    sudo pvs
    sudo vgs
    sudo lvs

Детальная информации по слоями:

    sudo pvdisplay
    sudo vgdisplay
    sudo lvdisplay

Создание pv слоя:

    sudo pvcreate /dev/sda6 /dev/sda7 

Можно перечислить подряд сколько угодно устройств, для каждого будет создан pv слой.


Создание vg слоя (пул памяти, который будет распределен между логическими томами и может состоять из нескольких физических разделов):

    sudo vgcreate vg_name /dev/sda6 /dev/sda7

Создание lv слоя:

    sudo lvcreate -L 80M -n lv_name vg_name
    sudo lvcreate -l 100%FREE -n lv_name vg_name
    
Когда lv слой создан, мы можем работать с ним как с обычным разделом.
Например, отформатируем его в файловую систему ext4, а затем примонтируем в /mnt:

    sudo mkfs.ext4 /dev/vg_name/lv_name
    sudo mount /dev/vg_name/lv_name /mnt/



LVM разделы могут быть трех типов:

* Линейные разделы (Linear Volume)
> Линейные разделы - это обычные LVM тома, они могут быть созданы как их одного, так и нескольких физических дисков. Например, если у вас есть два диска по 2 гигабайта, то вы можете их объединить и в результате получите один раздел LVM Linux, размером 4 гигабайта. По умолчанию используются именно линейные LVM разделы.
* Полосные разделы (Striped Volume)
> Полосные разделы очень полезны при больших нагрузках на жесткий диск. Здесь вы можете настроить одновременную запись на разные физические устройства, для одновременных операций, это может очень сильно увеличить производительность работы системы. Чтобы создать такой раздер нужно задать количество полос записи с помощью опции -i, а также размер полосы опцией -l. Количество полос не должно превышать количества физических дисков. `ssudo lvcreate -L 1G -i 2 -n logical_vol2 vol_grp1`.
     

* Зеркалированные разделы (Mirrored Volume)

> Зеркалированный том позволяет записывать данные одновременно на два устройства. Когда данные пишутся на один диск, они сразу же копируются на другой. Это позволяет защититься от сбоев одного из дисков. Если один из дисков испортится, то разделы LVM просто станут линейными и все данные по-прежнему будут доступны. Для создания такого раздела LVM Linux можно использовать команду: `sudo lvcreate -L 200M -m1 -n lv_mirror vol_grp1` .

Удаление LVM раздела:
    
     sudo lvremove /dev/vol-grp1/lv_mirror
     

Изменение размера LVM раздела:

    sudo lvextend -L+100M /dev/vol_grp1/logical_vol1

Для добавление используется знак + , а убавление места знак - после ключа -L.

Послед успешного изменения размера lvm раздела нужно проинициализировать эти изменения в системе через команду resize2fs:

    sudo resize2fs /dev/vol_grp1/logical_vol1
