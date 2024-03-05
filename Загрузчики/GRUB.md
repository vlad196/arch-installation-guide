
```bash
 sudo -u vlad paru -S grub grub-customizer
```
> [!INFO]
Внимание: os-prober в плане безопасности специально отключено в grub (В /etc/default/grub написано почему изначально os_prober отключён: "Probing for other operating systems is disabled for security reasons. ")
Поэтому просто самостоятельно добавляем windows в /etc/grub.d/, просто беря за основу саму записалось из os-prober

**Windows bootloader:**
```bash
cat << _EOF_ > /etc/grub.d/31_windows
#! /bin/sh
set -e

# Windows bootloader

cat << EOF
    menuentry 'Windows Boot Manager' --class windows {
        insmod part_gpt
        insmod fat
        search --no-floppy --fs-uuid --set=root B4DD-4624
        chainloader /EFI/Microsoft/Boot/bootmgfw.efi
    }
EOF
_EOF_
 && \
chmod 755 /etc/grub.d/31_windows
```
>[!Info]
>B4DD-4624 -- это UUID раздела где находиться загрузчик windows.
>Далее для linux это буде раздел с EFI контейнерами

#### Сами EFI Linux:
**Linux-lqx:**
```bash
cat << _EOF_ > /etc/grub.d/11_linux-lqx
#! /bin/sh
set -e

# Linux-lqx UKI

cat << EOF
    menuentry 'Arch Linux lqx' --class arch --class gnu-linux --class gnu {
        load_video
        set gfxpayload=keep
        insmod part_gpt
        insmod fat
        search --no-floppy --fs-uuid --set=root $NVME0N1P1
        chainloader /EFI/Linux/arch-linux-lqx.efi
    }
    submenu 'Дополнительные параметры для Arch Linux lqx' {
        menuentry 'Arch Linux Fallback lqx' --class arch --class gnu-linux --class gnu {
            load_video
            set gfxpayload=keep
            insmod part_gpt
            insmod fat
            search --no-floppy --fs-uuid --set=root $NVME0N1P1
            chainloader /EFI/Linux/arch-linux-lqx-fallback.efi
        }
    }
EOF
_EOF_
 && \
chmod 755 /etc/grub.d/11_linux-lqx
```
**Linux-lts:**
```bash
cat << _EOF_ > /etc/grub.d/12_linux-lts
#! /bin/sh
set -e

# Linux-lts UKI

cat << EOF
    menuentry 'Arch Linux lts' --class arch --class gnu-linux --class gnu {
        load_video
        set gfxpayload=keep
        insmod part_gpt
        insmod fat
        search --no-floppy --fs-uuid --set=root $NVME0N1P1
        chainloader /EFI/Linux/arch-linux-lts.efi
    }
    submenu 'Дополнительные параметры для Arch Linux lts' {
        menuentry 'Arch Linux Fallback lts' --class arch --class gnu-linux --class gnu {
            load_video
            set gfxpayload=keep
            insmod part_gpt
            insmod fat
            search --no-floppy --fs-uuid --set=root $NVME0N1P1
            chainloader /EFI/Linux/arch-linux-lts-fallback.efi
        }
    }
EOF
_EOF_
 && \
chmod 755 /etc/grub.d/12_linux-lts
```
**Перемещаем лишние скрипты в бэкап папку:**
```bash
mkdir /etc/grub.d/grub_backup && \
mv /etc/grub.d/10_linux /etc/grub.d/20_linux_xen /etc/grub.d/grub_backup
```
**Устанавливаем grub:**
```bash
grub-install --efi-directory=/efi --boot-directory=/efi --bootloader-id=GRUB --modules="tpm" --disable-shim-lock --debug
```
 Генерируем grub:
 ```bash
grub-mkconfig -o /efi/grub/grub.cfg 
```
**Подписываем файлы:**
```bash
sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI && sudo sbctl sign -s /efi/EFI/GRUB/grubx64.efi 
```
#### Тема:
**Скачиваем тему distro-grub-themes\\asrock и заходим в каталог и разархивируем:**
```bash
cd /home/vlad/bin && \
wget https://github.com/AdisonCavani/distro-grub-themes/raw/master/themes/asrock.tar && \
mkdir ./asrock && \
tar -C /home/vlad/bin/asrock -xvf /home/vlad/bin/asrock.tar
```
**Копируем в папку efi и редактируем настройки grub:**
```bash
cp -r /home/vlad/bin/asrock /efi/grub/themes/ && \
sed '/GRUB_THEME/s/^#//' -i /etc/default/grub && \
sed -i 's|GRUB_THEME="/path/to/gfxtheme"|GRUB_THEME=/efi/grub/themes/asrock/theme.txt|' /etc/default/grub && \
sed -i 's|GRUB_GFXMODE=auto|GRUB_GFXMODE=1920x1080|' /etc/default/grub
```

