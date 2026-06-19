# manual-image-overlay-offline-rootfs-injection
Рецепт для rootfs overlay через прямое монтирование образа

# rootfs overlay через прямое монтирование образа

Суть: `ds-rk3568-evb-sdcard.img` — это обычный диск-образ с разметкой GPT,
внутри которого лежат несколько партиций (loader, env, rootfs). Мы находим
loop-устройство, монтируем нужную партицию (rootfs, ext4) прямо как обычную
директорию в хост-системе, и копируем туда файлы — точно так же, как если
бы это была смонтированная флешка.

После этого образ становится самодостаточным: внутри уже лежит и базовая
система (ядро, u-boot, библиотеки), и наше приложение со всеми конфигами.
Прошивка такого образа на чистую (maskrom) плату одним действием через
`usb-upload-emmc-*.sh` сразу даёт полностью рабочий прибор — без отдельного
шага с распаковкой tar-архива после.

Это не то же самое, что **buildroot rootfs overlay** (`BR2_ROOTFS_OVERLAY`)
— тот механизм встраивает файлы автоматически на этапе `make`, при каждой
пересборке. Наш способ — разовая ручная операция над уже готовым `.img`
файлом, без пересборки buildroot. Это быстрее для разовой задачи, но нужно
повторять при каждой новой версии базового образа.

## Что нужно для работы

- Готовый базовый образ: `ds-rk3568-evb-sdcard.img`
- Архив с верхнеуровневой частью: `tmx38_control_<дата>.tar.gz`

## Пошаговая инструкция

### 1. Проверить структуру партиций образа

```bash
cd /home/work/rockchip/macrogroup/bsp/output/ds-rk35xx-evb/images/
sudo /sbin/fdisk -l ds-rk3568-evb-sdcard.img
```

Ожидаемый вывод (разметка GPT, три партиции):

```
Device                    Start     End Sectors  Size Type
ds-rk3568-evb-sdcard.img1    64    5183    5120  2,5M Linux filesystem
ds-rk3568-evb-sdcard.img2  5184    7231    2048    1M unknown
ds-rk3568-evb-sdcard.img3  7232 2464831 2457600  1,2G Linux root (ARM-64)
```

Партиция **3** (`Linux root (ARM-64)`) — это rootfs, именно туда нужно
копировать файлы. Партиции 1 и 2 — служебные (boot/env), их не трогаем.

> Если `fdisk` не находится в `PATH` — используйте полный путь
> `/sbin/fdisk`, пакет обычно уже установлен.

### 2. Создать loop-устройство и разобрать партиции

```bash
sudo /sbin/losetup -fP ds-rk3568-evb-sdcard.img
sudo /sbin/losetup -a
```

Вывод покажет имя устройства, например:

```
/dev/loop0: [2050]:26263510 (/home/.../ds-rk3568-evb-sdcard.img)
```

Флаг `-P` указывает ядру разобрать таблицу партиций внутри файла и создать
под-устройства автоматически:

```bash
ls /dev/loop0p*
# /dev/loop0p1  /dev/loop0p2  /dev/loop0p3
```

Если под-устройства не появились — форсировать пересчёт партиций:

```bash
sudo partprobe /dev/loop0
ls /dev/loop0p*
```

### 3. Смонтировать rootfs-партицию

```bash
sudo mkdir -p /mnt/rootfs_mount
sudo mount /dev/loop0p3 /mnt/rootfs_mount
ls /mnt/rootfs_mount
```

Ожидаемый вывод — обычная структура корня Linux:

```
bin boot dev etc home lib lib64 linuxrc lost+found media mnt opt proc
root run sbin srv sys tmp usr var
```

Если видите такую структуру — значит партиция смонтирована верно.

### 4. Распаковать архив приложения прямо в смонтированный rootfs

```bash
sudo tar -xvzf ~/SOVA2.0/tmx38_control_<дата>.tar.gz -C /mnt/rootfs_mount
```

Архив накатывается точно так же, как если бы это было сделано `busybox
tar -xvf - -C /` на самой плате — пути внутри архива (`./etc/...`,
`./root/...`, `./boot/...`) относительные, поэтому всё ложится в нужные
места внутри rootfs.

### 5. Проверить, что всё легло на место

```bash
ls -la /mnt/rootfs_mount/root/
# должны быть: TMX38_Control, media-probe, start_tmx38_control.sh

cat /mnt/rootfs_mount/etc/rkaiq_inputs.conf
# конфиг камер (будет перезаписан media-probe при первом старте — это ок)

cat /mnt/rootfs_mount/etc/xdg/weston/weston.ini
# секция [autolaunch] должна указывать на:
# path=/root/start_tmx38_control.sh
```

### 6. Отмонтировать и освободить loop-устройство

```bash
sudo umount /mnt/rootfs_mount
sudo /sbin/losetup -d /dev/loop0
```

После этого шага **обязательно** отмонтировать перед передачей файла —
иначе изменения могут быть не до конца сброшены на диск, и итоговый
`.img` окажется неконсистентным.

### 7. Передать готовый образ

Файл `ds-rk3568-evb-sdcard.img` теперь самодостаточен

Для полной прошивки чистой платы:


```bash
./usb-upload-emmc-ds-rk3568-evb.sh
```

Скрипт одной командой зальёт весь образ (включая rootfs с нашим
приложением) на eMMC через USB/fastboot — после этого плата сразу
полностью рабочая, без отдельных шагов.

