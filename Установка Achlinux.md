## Обозначения:
### Машина-клиент -- машина, на которую устанавливают систему
### Машина-настройщик -- машина, через которую устанавливают систему по ssh

# Часть 0. Настройка UEFI
**1) Если решили заново настроить и нет уже готовых ключей для подписи, то убеждаемся, что Secure-boot сброшен и ключи пусты (Не вернувшиеся в дефолтное состояние, а именно пустые)**
**2) Проверяем, что для не подписанного arch iso был отключён Secure-boot**
**3) Проверяем, что TPM включён**
# Часть 1. Настройка ssh

>[!Note]
>Установка через ssh позволяет сразу копировать готовые команды

**На машине-клиенте устанавливаем пароль:**
```bash
passwd
```
**На машине-клиент узнаём ip его локальный ip(Будет примерно 192.168.1.111):**
```bash
ip -br a
```
**Далее на машине-настройщике подключаемся по ssh к машине-клиенту**
```bash
ssh ssh -o ServerAliveInterval=29 root@<ip машины-клиента, который узнали выше>
```
>[!NOTE]
>Добавил ServerAliveInterval=29, чтобы не было отключения при бездействии.
# Часть 2. Подготовка дисков:
## Удаление данных с диска:
```bash
wipefs -a /dev/nvme0n1
```

## Создание разметки и разделов:
**Размечаем диск в GPT:**
```bash
parted /dev/nvme0n1 mklabel gpt
```
**Создаём 2 раздела:**
```bash
parted /dev/nvme0n1 mkpart '"EFI system partition"' fat32 2048s 2GiB && \
parted /dev/nvme0n1 mkpart '"swap partition"' 2GiB 26GiB && \
parted /dev/nvme0n1 mkpart '"system partition"' 26GiB 100%
```
>[!NOTE]
>Очень важно, следите за размером секторов на диске! От этого зависит скорость доступа к диску. Если при создании раздела не соблюсти кратность сектора, то контроллер будет чаще обращаться к ячейкам, а значит увеличится и время доступа к данным. В данном случае, минимальный разрешённый сектор у меня 2048s(секторов).
>Дополнительно тут: https://wiki.archlinux.org/title/Parted_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)#%D0%92%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5

>[!NOTE]
>Ещё интересная вещь, это указание размера разделов в процентах. Она очень удобна, когда нужно указать раздел от начала 0% или до конца 100%

**Назначаем флаги для разделов (необязательно, но пусть будет):**
```bash
parted /dev/nvme0n1 set 1 esp on && \
parted /dev/nvme0n1 set 2 swap on
```
**Форматирование раздела под efi:**
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```
## LUKS шифрование разделов:
LUKS шифрование даёт нам раздел, который полностью закрыт для просмотра, делая из раздела, по сути один большой файл. Его структура похожа на структуру обычного диска с разметкой диска, только вместо разметки диска, у нас заголовок LUKS и ещё есть отдельный раздел для хранения ключей, за счёт чего в LUKS  можно удобно менять пароли (в отличии от того же dm-decrypt, где ключ выбирается один на все файлы)
Подробнее о моём варианте шифрования можно почитать тут:
https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
**Проверяем модули на работоспособность:**
```bash
modprobe dm-crypt && \
modprobe dm-mod
```
**Шифруем swap раздел в A 512:**
```bash
cryptsetup --verbose luksFormat --key-size 512 --hash sha512 /dev/nvme0n1p2
```
**Шифруем root раздел в A 512:**
```bash
cryptsetup --verbose luksFormat --key-size 512 --hash sha512 /dev/nvme0n1p3
```
**Открываем зашифрованные разделы:**
```bash
cryptsetup luksOpen /dev/nvme0n1p2 swap && \
cryptsetup --allow-discards luksOpen /dev/nvme0n1p3 root
```
**Экспортируем UUID дисков в переменные:**
```bash
export NVME0N1P1=$(lsblk -dno UUID /dev/nvme0n1p1) \
NVME0N1P2=$(lsblk -dno UUID /dev/nvme0n1p2) \
NVME0N1P3=$(lsblk -dno UUID /dev/nvme0n1p3)
```
**Экспортируем адреса зашифрованных контейнеров:**
```bash
export ROOT=/dev/mapper/root \
SWAP=/dev/mapper/swap
```
## Создание файловых систем в томах:
**Создание файловой системы swap:**
```bash
mkswap -L swap $SWAP
```
**Создание файловой системы f2fs:**
```bash
mkfs.f2fs -l "Arch Linux" -O extra_attr,inode_checksum,sb_checksum,compression  $ROOT
```
>[!NOTE]
>Для подробностей нужно идти на Archwiki, но тут, как минимум включена полезная компрессия
>Подробнее тут: https://wiki.archlinux.org/title/F2FS#Compression
>
## Монтирование разделов:
**Обновление информации о дисках:**
```bash
systemctl daemon-reload
```
**Монтирование корневого раздела:**
```bash
mount -o compress_algorithm=zstd:6,compress_chksum,atgc,gc_merge,lazytime $ROOT /mnt
```
>[!NOTE]
>Раздел смонтирован с рекомендуемыми Archwiki параметрами
>Подбробнее тут: https://wiki.archlinux.org/title/F2FS#Recommended_mount_options

**Монтирование swap:**
```bash
swapon $SWAP
```
**Монтирование efi раздела:**
```bash
mount --mkdir -o uid=0,gid=0,fmask=0137,dmask=0027  /dev/nvme0n1p1 /mnt/efi
```
>[!NOTE]
>Делаем маску 0077, ради того, чтобы из под обычного пользователя не было доступа к этому разделу. Собственно, systemd-boot при установке про это и говорит
>``` bash
>!Mount point '/boot' which backs the random seed file is world accessible, which is a security hole! !
>! Random seed file '/boot/loader/random-seed' is world accessible, which is a security hole! !
>```

**Мой дополнительный диск:**
```bash
mount --mkdir /dev/sdb /mnt/mnt/sdb
```
# Часть 3. Установка до подмены корневого раздела:

**Подбор зеркал:**
Зачастую, стандартный подбор зеркал является неоптимальным. Поэтому с помощью reflector зеркала отсортируем по скорости и типу и сохраним их
```bash
reflector --verbose -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
```
**Устанавливаем базовые пакеты:**
```bash
pacstrap -K /mnt base base-devel git vi neovim mkinitcpio reflector
```
**Генерирование FSTAB:**
```bash
genfstab -U /mnt > /mnt/etc/fstab
```
**Переход в CHROOT:**
```bash
arch-chroot /mnt
```
# Часть 4. Настройка в подменённом корневом разделе:
### Установка времени:
**Установка своего часового пояса:**
```bash
ln -sf /usr/share/zoneinfo/Europe/Kaliningrad /etc/localtime
```
>[!NOTE]
>Здесь выбираем собственный часовой пояс
**Синхронизация:**
```bash
hwclock --systohc
```
### Установка локалей:
**Установка eng локали:**
```bash
sed '/en_US.UTF-8 UTF-8/s/^#//' -i /etc/locale.gen
```
**Установка ru локали:**
```bash
sed '/ru_RU.UTF-8 UTF-8/s/^#//' -i /etc/locale.gen
```
**Генерация:**
```bash
locale-gen
```
**Устанавливаем язык по умолчанию:**
```bash
echo LANG=ru_RU.UTF-8 >> /etc/locale.conf
```
**Установка шрифтов**
```bash
cat << _EOF_ > /etc/vconsole.conf
KEYMAP=ruwin_alt_sh-UTF-8
FONT=ter-v16n
_EOF_
```
>[!NOTE]
>ruwin_alt_sh-UTF-8 раскладка, в отличии от раскладки, предлагаемой в "Installation guide" даёт возможность в режиме терминала менять раскладку через alt + shift
>ter-v16n обычно, находиться внутри самого ядра. Есть проблема с тем, что после `splash` не загружаются внешние шрифты, поэтому выбрал из ядра тот, который поддерживает utf-8.
#### Создание хоста:
**Создаём имя хоста:**
```bash
echo "ArchLinux" >> /etc/hostname
```

### Создание пользователя:
**Создание пароля для root:**
```bash
passwd
```
**Добавляем пользователя:**
```bash
useradd -mG wheel vlad
```
**Делаем бэкап файла:**
```bash
cp /etc/sudoers /etc/sudoers.backup
```
**Редактируем сам sudoers:**
```bash
sed '/\%wheel ALL=(ALL:ALL) ALL/s/^# //' -i /etc/sudoers
```
**Проверяем на правильность:**
```bash
visudo -c /etc/sudoers
```
**Если всё хорошо, то удаляем бэкап:**
```bash
rm /etc/sudoers.backup
```
**Добавляем пароль для пользователя:**
```bash
passwd vlad
```

## Настройка компилятора и пакетного менеджера:
**Для мнимой производительности для нашего пользователя переназначаем флаги GCC:**
```bash
sudo -u vlad cat << _EOF_ > /home/vlad/.makepkg.conf
CFLAGS="-march=native -mtune=native -O2 -pipe -fno-plt -fexceptions \\
-Wp,-D_FORTIFY_SOURCE=3 -Wformat -Werror=format-security \\
-fstack-clash-protection -fcf-protection"
CXXFLAGS="$CFLAGS -Wp,-D_GLIBCXX_ASSERTIONS"
RUSTFLAGS="-C opt-level=3 -C target-cpu=native -C link-arg=-z -C link-arg=pack-relative-relocs"
MAKEFLAGS="-j$(nproc) -l$(nproc)"
_EOF_
```
>[!NOTE]
>Все эти флаги взяты отсюда:
>https://ventureo.codeberg.page/v2022.07.01/source/generic-system-acceleration.html#makepkg-conf

### PARU:
Отличительной особенностью PARU является одновременно и то что он написан на Rust и то что он позволяет достаточно удобно работать с PKGBUILD
**Скачиваем PARU и входим в его каталог:**
```bash
sudo -u vlad git clone https://aur.archlinux.org/paru.git /home/vlad/bin/paru
```
**Создаём пакет, устанавливаем его и переходим обратно в корень**
```bash
(cd /home/vlad/bin/paru/ && sudo -u vlad makepkg -si)
```

**Добавляем возможность редактирования в paru:**
```bash
sed '/\[bin\]/s/^#//' -i /etc/paru.conf && \
sed '/FileManager/s/^#//' -i /etc/paru.conf && \
sed '/FileManager/s/vifm$/nvim/' -i /etc/paru.conf
```
**Заставляем paru держать таймер истечения действия пароля до полного выполнения работы:**
```bash
sed '/SudoLoop/s/^#//' -i /etc/paru.conf
```
### Добавление своего стека репозиториев:
**Делаем копию pacman.conf:**
```bash
cp /etc/pacman.conf /etc/pacman.conf.backup
```
**В /etc/pacman.conf раcкомментируем multilib и добавояем комментарий:**
```bash
sed '/\[multilib\]/{s/^#//;n;s/^#//;}' -i /etc/pacman.conf && \
sed '/\[core-testing\]/i\
\
# Default repositories\
' -i /etc/pacman.conf
```
**Раcкомментируем параллельные загрузки и цветной вывод:**
```bash
sed '/ParallelDownloads =/s/^#//' -i /etc/pacman.conf && \
sed '/Color/s/^#//' -i /etc/pacman.conf
```

**Добавляем репозитории которые Вам нужны:**
Объявление через комментарий о участке со своими репозиториями
```bash
sed -i "/# after the header, and they will be used before the default mirrors./{
n
n
a\\
#Custom\ repositories\n
}" -i /etc/pacman.conf
```
CachyOS репозитории (Репозитории скомпилированный под 86-64-v3 архитектуру)
Обязательно! Перед добавлением в pacman.conf добавляем ключи с их сайта:
```bash
pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com && \
pacman-key --lsign-key F3B607488DB35A47
```
Устанавливаем необходимые пакеты:
```bash
pacman -U 'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-keyring-20240331-1-any.pkg.tar.zst' \
    'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-mirrorlist-22-1-any.pkg.tar.zst' \
    'https://mirror.cachyos.org/repo/x86_64/cachyos/cachyos-v3-mirrorlist-22-1-any.pkg.tar.zst' \
    'https://mirror.cachyos.org/repo/x86_64/cachyos/pacman-7.0.0.r7.g1f38429-1-x86_64.pkg.tar.zst'
```

Уже потом добавляем в pacman.conf:
```bash
sed '/# Default repositories/i\
\# cachyos repos\
\[cachyos-v3\]\
Include = /etc/pacman.d/cachyos-v3-mirrorlist\
\
\[cachyos-extra-v3\]\
Include = /etc/pacman.d/cachyos-v3-mirrorlist\
\
\[cachyos\]\
Include = /etc/pacman.d/cachyos-mirrorlist\
' -i /etc/pacman.conf
```

>[!NOTE]
>Репозитории написаны в порядке моего приоритета. С каждым новым вводом, следующий репозиторий будет записываться в конец списка

>[!NOTE]
>Следите за очерёдностью репозиториев. Репозитории расположенные вверху имеют приоритет перед нижними

### Обновление ключей и репозиториев:
Обновляем зеркала по скорости уже в chroot:
```bash
reflector --verbose -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
```
**Инициализация:**
```bash
sudo -u vlad paru -Sy archlinux-keyring && sudo -u vlad paru -Su
```
### Добавление пакетов:
**Скачиваем необходимые пакеты. Микрокод, f2fs пакеты, менеджер сети, менеджер efiboot, lvm2 и дополнительные шрифты:**
```bash
sudo -u vlad paru -S --needed wget man f2fs-tools amd-ucode \
efibootmgr networkmanager bluez pipewire  \
lib32-pipewire noto-fonts-cjk ttf-hannom \
wl-clipboard terminus-font xdg-utils mailcap
```

```bash
sudo -u vlad paru -S --asdeps --needed bat devtools lib32-dbus pipewire-pulse pipewire-jack lib32-pipewire-jack lib32-libavtp lib32-libsamplerate lib32-libpulse lib32-speexdsp lib32-pipewire-v4l2
```
## Первоначальная настройка:
### Включение и настройка сети и bluetooth:
>[!NOTE]
> Если не включить экспериментальные функции для bluetooth, в journald будут предупреждения о отсутствии поддержки некоторых плагинов:
> ```bash
> src/plugin.c:plugin_init() System does not support micp plugin
> src/plugin.c:plugin_init() System does not support vcp plugin
> src/plugin.c:plugin_init() System does not support mcp plugin
> src/plugin.c:plugin_init() System does not support bass plugin
> src/plugin.c:plugin_init() System does not support bap plugin
>```
> Проблем от отсутствия поддержки плагинов я не заметил, поэтому настройка необязательна.

**Включаем экспериментальные функции bluetooth, чтобы не было предупреждений в journald:**
```bash
sed '/Enables D-Bus experimental interfaces/{n;n;s/^#//;s/false/true/;}' -i /etc/bluetooth/main.conf
sed '/Enables D-Bus testing interfaces/{n;n;s/^#//;s/false/true/;}' -i /etc/bluetooth/main.conf
sed '/KernelExperimental/{s/^#//;s/false/true/;}' -i /etc/bluetooth/main.conf
```
**Включаем юниты менеджера сети и bluetooth:**
```bash
systemctl enable NetworkManager.service && \
systemctl enable bluetooth.service
```
### Ananicy CPP и готовые правила
Установка:
```bash
paru -S ananicy-cpp cachyos-ananicy-rules
```
Запуск:
```bash
systemctl enable ananicy-cpp.service
```

### systemd-oomd
Запуск:
```bash
systemctl enable systemd-oomd.service
```
### irqbalance
Установка:
```bash
paru -S irqbalance
```
Запуск:
```bash
systemctl enable irqbalance.service
```

### ccache
Установка:
```bash
paru -S ccache
```
Редактируем makepkg.conf:
```bash
sudo -u vlad cat << _EOF_ >> /home/vlad/.makepkg.conf
BUILDENV=(!distcc color ccache check !sign)
_EOF_
```
### Отключение многоступенчатого включения дисков

```bash
cat << _EOF_ > /etc/modprobe.d/30-ahci-disable-sss.conf
options libahci ignore_sss=1
_EOF_
```

### Редактирование MKINITCPIO:
**Копируем mkinitcpio.conf в mkinitcpio.conf.d:**
```bash
cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.d/mkinitcpio.conf
```

**Редактируем hook mkinitcpio и включаем туда модули systemd и после block хуки sd encrypt. Меняем шифрование:**
```bash
sed -i '/COMPRESSION="lz4"/s/^#//' -i /etc/mkinitcpio.conf.d/mkinitcpio.conf &&\
sed -i '/COMPRESSION_OPTIONS=()/s/^#//' -i /etc/mkinitcpio.conf.d/mkinitcpio.conf &&\
sed -i 's/^\(COMPRESSION_OPTIONS=(\))$/\1-9)/' /etc/mkinitcpio.conf.d/mkinitcpio.conf  && \
sed -i '/^HOOKS=/ s/keymap consolefont/sd-vconsole/' /etc/mkinitcpio.conf.d/mkinitcpio.conf && \
sed -i "/^HOOKS=/ s/\(block\)\(.*\)$/\1 sd-encrypt\2/" /etc/mkinitcpio.conf.d/mkinitcpio.conf
```
>[!NOTE]
>Когда есть systemd, хук resume не нужен
>https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs
>
>Про шифрование https://ventureo.codeberg.page/source/boot.html#initramfs

**Добавляем /etc/crypttab.initramfs: **
```bash
cat << _EOF_ > /etc/crypttab.initramfs
# Mount /dev/mapper/swap re-encrypting it with a fresh key each reboot
swap UUID=$NVME0N1P2 none timeout=180,tpm2-device=auto
# Mount /dev/mapper/root with the key from TPM
root UUID=$NVME0N1P3 none timeout=180,tpm2-device=auto,discard
_EOF_
```
### Установка SECURE BOOT:
```bash
sudo -u vlad paru -S  sbctl
```
**Проверяем, что secure boot в setup mode: Enabled (нужно удалить старые ключи из биоса):**
```bash
sbctl status
```
**Создаём свои ключи:**
```bash
sbctl create-keys
```
**Создаём свои ключи и добавляем ключи Microsoft (для поддержки windows и вшитых устройств у некоторых материнок):**
```bash
sbctl enroll-keys --microsoft
```
>[!NOTE]
>Если secure boot не очищенный, UEFI не даст накатить ключи. Если до этого Secure boot был забыт, можно пока пропустить этот шаг. При перезагрузке удалить ключи и уже потом накатывать
## Ядра:
### Установка UKI:

**Добавить опции основного ядра в cmdline:**
```bash
cat << _EOF_ > /etc/kernel/cmdline
options page_alloc.shuffle=1 root=$ROOT rootflags=atgc resume=$SWAP rw
_EOF_
```
**Добавим опции для fallback (В итоге, он как запасной имеет минимальные для загрузки параметры, например будет в дальнейшем без plymouth параметров):**
```bash
cat << _EOF_ > /etc/kernel/cmdline-base
options page_alloc.shuffle=1 root=$ROOT rootflags=atgc rw
_EOF_
```
>[!NOTE]
>**page_alloc.shuffle=1** - Этот параметр рандомизирует свободные списки
> распределителя страниц. Улучшает производительность при работе с ОЗУ с очень быстрыми накопителями (NVMe, Optane). Подробнее [тут](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e900a918b0984ec8f2eb150b8477a47b75d17692).
>Доп. информация здесь: https://ventureo.codeberg.page/source/kernel-parameters.html
>
>**root=** - опция, указывающая какой раздел грузить как root (в данном случае это логический диск)
>resume= -опция, показывающий системе swap раздел, необходимый при сне.
>
>**rootflags=atgc** - опцию, которую я не расшифровал, но тут написано зачем оно:https://wiki.archlinux.org/title/F2FS#Remounting_impossible_with_some_options
>
>**rw** - разрешение на чтение запись раздела

> [!NOTE]
> Раньше был параметр `resume=$SWAP` для пробуждения из гибернизации, но компоненты systemd научились работать без этого указателя.
>
> Если стоит busybox, необходимо вернуть этот параметр.

**Добавляем в переменные нужные ядра:**
```bash
export MAIN_KERNEL=linux-cachyos
```

**Добавляем пресеты для разных UKI:**
```bash
cat << _EOF_ > /etc/mkinitcpio.d/$MAIN_KERNEL.preset
# mkinitcpio preset file for the '$MAIN_KERNEL' package

ALL_config="/etc/mkinitcpio.conf.d/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-$MAIN_KERNEL"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf.d/mkinitcpio.conf"
#default_image="/boot/initramfs-$MAIN_KERNEL.img"
default_uki="/efi/EFI/Linux/arch-$MAIN_KERNEL.efi"
default_options="--cmdline /etc/kernel/cmdline"

#fallback_config="/etc/mkinitcpio.conf.d/mkinitcpio.conf"
#fallback_image="/boot/initramfs-$MAIN_KERNEL-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-$MAIN_KERNEL-fallback.efi"
fallback_options="--cmdline /etc/kernel/cmdline-base"
_EOF_
```

Перед установкой, возможно, захочется не компилировать некоторые ядра, а поставить нужный репозиторий:

[Linux-mainline](/Ядра/Репозитории%20ядер.md#Linux-mainline)

[Linux-lqx](/Ядра/Репозитории%20ядер.md#Linux-lqx)

[Linux-cachyos](/Ядра/Репозитории%20ядер.md#Linux-cachyos)

**Создать директории в efi, если их нет**
```bash
mkdir -p /efi/EFI && \
mkdir -p /efi/EFI/LINUX
```

**Устанавливаем ядро и нужные компоненты:**
```bash
sudo -u vlad paru -S --needed $MAIN_KERNEL  $MAIN_KERNEL-headers mkinitcpio-firmware
```

```bash
sudo -u vlad paru -S --needed --asdeps wireless-regdb linux-firmware  modprobed-db uksmd
```

> [!NOTE]
>Для того, чтобы mkinitcpio не делал предупреждений о отсутствующих модулях, я поставил
>этот пакет(Но это не обязательно т.к. оно ни на что не влияет):
>https://wiki.archlinux.org/title/Mkinitcpio#Possibly_missing_firmware_for_module_XXXX

## **Настройки для видеокарты NVIDIA:**
Если видеокарта не имеет type-c, чтобы избавиться от ошибок можно его замутить
**Добавляем в modprobe.d:**
```bash
cat << _EOF_ > /etc/modprobe.d/blacklist_i2c.conf
blacklist i2c_nvidia_gpu
#
# If the video card does not have type-c, and support is included in the drivers, then you can mute it
_EOF_
```
### [NVIDIA](/Драйвера/NVIDIA.md)
### [Nouveau](/Драйвера/Nouveau.md)

## Загрузчики:
### [Systemd-boot](/Загрузчики/Systemd-boot.md)
### [GRUB](/Загрузчики/GRUB.md)
### [Запись UKI в UEFI](/Загрузчики/Запись-UKI-в-UEFI.md)
## Пересобираем ядра уже в EFI:
```bash
mkinitcpio -P
```
### Подпись файлов efi созданными ключами:
**Подписываем Windows файлы:**
```bash
sbctl sign -s /efi/EFI/Boot/bootx64.efi
```

## UEFI приложения:
### memtest86:
**Установка:**
```bash
sudo -u vlad paru -S memtest86-efi
```
**Запускаем скрипт установки и следуем инструкциям:**
```bash
memtest86-efi --install
```
**Подписываем:**
```bash
sbctl sign -s /efi/EFI/memtest86/memtestx64.efi
```
**Добавлляем папку pacman.d/hooks**
```bash
mkdir -p /etc/pacman.d/hooks
```

**Добавляем триггер для подписи memtest86 при его обновлении:**
```bash
cat << _EOF_ >> /etc/pacman.d/hooks/sign-memtest86-secureboot.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = /efi/EFI/memtest86/memtestx64.efi

[Action]
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
Depends = sbctl
_EOF_
```
### fwupd:
**Установка:**
```bash
sudo -u vlad paru -S --needed fwupd
```
**Подписываем:**
```bash
sbctl sign -s /usr/lib/fwupd/efi/fwupdx64.efi
```
**Добавляем триггер для подписи fwupd при его обновлении:**
```bash
cat << _EOF_ >> /etc/pacman.d/hooks/sign-fwupd-secureboot.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = /usr/lib/fwupd/efi/fwupdx64.efi

[Action]
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
Depends = sbctl
_EOF_
```
## Постзагрузка:
>[!Warning]
>Часть инструкций ломает внешние шрифты. Пока не добавлять при установке
### Plymouth:
**В cmdline добавить:**
```bash
sed -i -e 's/$/ loglevel=3 quite splash rd.udev.log_priority=3 vt.global_cursor_default=0/' /etc/kernel/cmdline
```
**В hook в mkinitcpio после udev вставить plymouth:**
```bash
sed -i "/^HOOKS=/ s/\(systemd\)\(.*\)$/\1 plymouth\2/" /etc/mkinitcpio.conf.d/mkinitcpio.conf
```
> [!NOTE]
> У nvidia plymouth появляется довольно поздно и это нормально.
> Всё из-за поздней загрузки kms
### Настройка plymouth:
**Установка пакетов:**
```bash
sudo -u vlad paru -S plymouth plymouth-theme-arch-bgrt
```
**Выбираем тему arch-bqrt:**
```
plymouth-set-default-theme -R arch-bgrt
```
## Начало настройки рабочего окружения:

#### Украшаем окно приветствия:
```bash
cat << _EOF_ >> /etc/issue
 \e[H\e[2J
           \e[1;36m.
          \e[1;36m/#\
         \e[1;36m/###\      \e[1;37m               #     \e[1;36m| *
        \e[1;36m/p^###\     \e[1;37m a##e #%" a#"e 6##%  \e[1;36m| | |-^-. |   | \ /
       \e[1;36m/##P^q##\    \e[1;37m.oOo# #   #    #  #  \e[1;36m| | |   | |   |  X
      \e[1;36m/##(   )##\   \e[1;37m%OoO# #   %#e" #  #  \e[1;36m| | |   | ^._.| / \ \e[0;37mTM
     \e[1;36m/###P   q#,^\
    \e[1;36m/P^         ^q\ \e[0;37mTM
_EOF_
```
### Avahi

>[!NOTE]
Компонент тоже важный. Avahi устанавливается неявно вместе с pipewire, поэтому, отмечаю его отдельно, чтобы не потерялcя
Что это: Avahi is a free Zero-configuration networking (zeroconf) implementation, including a system for multicast DNS/DNS-SD service discovery. It allows programs to publish and discover services and hosts running on a local network with no specific configuration. For example you can plug into a network and instantly find printers to print to, files to look at and people to talk to. It is licensed under the GNU Lesser General Public License (LGPL).

>[!Что он делает:]
Avahi provides local hostname resolution using a "hostname.local" naming scheme.

**Установка:**
```bash
sudo -u vlad paru -S --needed avahi nss-mdns
```

**Включаем avahi-daemon.service:**
```bash
systemctl enable avahi-daemon.service
```

>[!NOTE]
Systemd-resolved конфликтует с Avahi в mDNS, поэтому отключаем у systemd-resolved mDNS

```bash
cat << _EOF_ >> /etc/systemd/resolved.conf.d/disable-mDNS.conf
[Resolve]
MulticastDNS=false
_EOF_
```

### AppArmor
**Устанавливаем сам apparmor (audit тоже качает ):**
```bash
sudo -u vlad paru -S apparmor
```
**Запускаем юнит apparmor:**
```bash
systemctl enable apparmor.service
```
>[!NOTE]
Прежде всего нужно убедиться, что ядро поддерживает apparmor

**Прописываем параметры ядра:**
```bash
sed -i -e 's/$/ lsm=landlock,lockdown,yama,integrity,apparmor,bpf audit=1 audit_backlog_limit=8192/' /etc/kernel/cmdline
```
>[!NOTE]
>audit_backlog_limit=8192 используется для предотвращения ошибки:
>'''bash
>audit: kauditd hold queue overflow
>'''

**Обновляем и подписываем ядро:**
```bash
mkinitcpio -P
```
#### Для работы уведомлений от Apparmor нужно:
**Устанавливаем пакеты:**
```bash
sudo -u vlad paru -S --asdeps --needed python-notify2 python-psutil
```
**Создаём группу аудита:**
```bash
groupadd --system audit
```
**Добавляем юзера в группу аудита:**
```bash
usermod vlad -aG audit
```
**Добавляем группу аудита в конфига аудита:**
```bash
sed '/log_group = root/s/root/audit/' -i /etc/audit/auditd.conf
```
**Создаём папку автостарт, если нет:**
```bash
sudo -u vlad mkdir -p /home/vlad/.config/autostart
```
**Добавляем .desktop файл для уведомления:**
```bash
sudo -u vlad bash -c 'cat << _EOF_ >> /home/vlad/.config/autostart/apparmor-notify.desktop
[Desktop Entry]
Type=Application
Name=AppArmor Notify
Comment=Receive on screen notifications of AppArmor denials
TryExec=aa-notify
Exec=aa-notify -p -s 1 -w 60 -f /var/log/audit/audit.log
StartupNotify=false
NoDisplay=true
_EOF_'
```
**Запускаем аудита:**
```bash
systemctl enable auditd.service
```

**Создаём файл syslog, если его нет:**
```
touch /var/log/syslog
```
**Сниманием комментарий с write-cahe:**
```bash
sed '/write-cache/s/^#//' -i /etc/apparmor/parser.conf
```
## Графические окружения:
> [!NOTE]
> На этом этапе можно начинать устанавливать графическое окружение желательно выбрать одно из окружений, т.к. при параллельном использовании одно окружение может влиять на другое

[Gnome](/Среды%20рабочего%20стола/Gnome.md)

[KDE](/Среды%20рабочего%20стола/KDE.md)
## XDG-USER-DIRS
Множество программ используют спецификацию XDG, и для таких программ, как файловый менеджер можно прямо указать расположение типовых каталогов (изображения,  документы и т.п.) Это неплохая альтернатива мягким ссылкам

**Установка:**
```bash
sudo -u vlad paru -S --needed xdg-user-dirs
```

**Указываем папки для типовых каталогов:**
```bash
sudo -u vlad mkdir -p /home/vlad/.config && \
sudo -u vlad cat << _EOF_ > /home/vlad/.config/user-dirs.dirs
# This file is written by xdg-user-dirs-update
# If you want to change or add directories, just edit the line you're
# interested in. All local changes will be retained on the next run.
# Format is XDG_xxx_DIR="$HOME/yyy", where yyy is a shell-escaped
# homedir-relative path, or XDG_xxx_DIR="/yyy", where /yyy is an
# absolute path. No other format is supported.
#
XDG_DESKTOP_DIR="/mnt/sdb/Рабочий стол"
XDG_DOWNLOAD_DIR="/mnt/sdb/Загрузки"
XDG_TEMPLATES_DIR="$HOME/Шаблоны"
XDG_PUBLICSHARE_DIR="$HOME/Общедоступные"
XDG_DOCUMENTS_DIR="/mnt/sdb/YandexDisk/Компьютер SURFACE-BOOK/Документы"
XDG_MUSIC_DIR="/mnt/sdb/Музыка"
XDG_PICTURES_DIR="/mnt/sdb/YandexDisk/Компьютер SURFACE-BOOK/Изображения"
XDG_VIDEOS_DIR="/mnt/sdb/Видео"
_EOF_
```

### Шрифты:
```bash
sudo -u vlad paru -S --needed ttf-ubuntu-nerd ttf-spacemono ttf-meslo-nerd-font-powerlevel10k
```

### Trim:
#### Непрерывный Trim:
В F2FS по умолчанию включён f2fs, который ведёт себя как непрерывный со своими особенностями

#### Периодический Trim:
**В /etc/fstab добавить nodiscard:**
```bash
genfstab -U / >> /etc/fstab
```
**Включение, старт и вывод периодического Trim:**
```bash
systemctl enable fstrim.service && \
systemctl start fstrim.service && \
systemctl status fstrim.service
```
> [!NOTE]
> Естественно, если стоит f2fs, то включать периодический Trim не стоит. У f2fs есть свой Trim

### ssh
**Установка**:
```bash
sudo -u vlad paru -S --needed openssh
```
%%
TODO: надо сюда добавить настройки по безопасности
%%

**Включаем юнит:**
```bash
systemctl enable sshd.service
```
### Перезагружаемся
> [!NOTE]
Обязательно проверяем, что UKI файлы на 100% собраны и без ошибок
Так как в этап создания initramfs теперь подвязаны драйвера nvidia и подпись файлов для secure boot,
то теперь (а логичней сделать отдельный хук для этого) нужно чтобы хуки от nvidia и подписи срабатывали
Нужно в конце сгенерировать mkinitcpio и подписать ядра:

```bash
mkinitcpio -P
```
**Выходим из chroot:**
```bash
exit
```
**Отмонтируем все тома и диски если LVM:**
```bash
umount -Rv /mnt &&\
swapoff /dev/mapper/swap &&\
cryptsetup close /dev/mapper/root && \
cryptsetup close /dev/mapper/swap
```

**Перезагружаемся:**
```bash
reboot
```

>[!Warning]
>Включаем secure boot и TPMtrusted compute в UEFI

---
# Всё что ниже нужно делать уже после перезагрузки системы
___
# Часть 5. Настройка после загрузки со своего ядра:
## Повторное подключение по ssh:

**На машине-клиент снова узнаём ip его локальный ip(Будет примерно 192.168.1.111):**
```bash
ip -br a
```
**Далее на машине-настройщике подключаемся по ssh к машине-клиенту:**
```bash
ssh vlad@<ip машины-клиента, который узнали выше>
```
## Переменные:
>[!NOTE]
>Они нам тут ещё пригодятся

**Экспортируем UUID дисков в переменные:**
```bash
export NVME0N1P1=$(lsblk -dno UUID /dev/nvme0n1p1) \
NVME0N1P2=$(lsblk -dno UUID /dev/nvme0n1p2) \
NVME0N1P3=$(lsblk -dno UUID /dev/nvme0n1p3) \
ROOT=/dev/mapper/root \
SWAP=/dev/mapper/swap
```
### Проверка Apparmor:
**Проверяем, работает ли aa-notify (должно быть хоть одно выполнение команды):**
```bash
pgrep -ax aa-notify
```

## Настройка LUKS ЧЕРЕЗ TPMtrusted compute
>[!NOTE]
>Помимо systemd-cryptenroll есть ещё Clevis, но он в итоге себя показал намного быстрее при загрузке
### Systemd-cryptenroll:
**Устанавливаем необходимые пакеты:**
```bash
paru -S --needed tpm2-tss tpm2-tools
```
**Проверяем распознаёт ли linux tpm модуль(Если при загрузке системы появляется ошибка tpm, то ничего страшного):**
```bash
test -c /dev/tpm0 && echo OK || echo FAIL
```
> [!NOTE]
**Если нет, то проверяем включён ли TPM модуль в материнке:**

**Создаём копию заголовка (Её лучше сразу на какую-то флешку перекинуть):**
```bash
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file /mnt/sdb/header-nvme0n1p3.img
```
%%
!!! Проверить просто sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 . Должно теперь работать без указания устройства
%%
**Привязываем luks к systemd-cryptenroll и внедряем в tpm ключ:**
```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme0n1p2
```
```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme0n1p3
```
**Проверяем всё ли записалось:**
```bash
sudo cryptsetup luksDump /dev/nvme0n1p3
```
**Обновляем ядра и подписываем их:**
```bash
sudo mkinitcpio -P
```

## Настойка подкачки отключения упреждённого чтения:

```bash
sudo bash -c "cat << __EOF__ > /etc/sysctl.d/99-sysctl.conf
vm.swappiness=100
vm.page-cluster=0
__EOF__"
```

## Настройка кэша VFS:

```bash
sudo bash -c "cat << __EOF__ > /etc/sysctl.d/99-vfs.conf
vm.vfs_cache_pressure=50
__EOF__"
```

## Установка и запуск планировщика, если ядро с sched-ext
```bash
paru -Sy scx-scheds && \
sudo systemctl enable --now scx
```

## Переменные для wayland

> [!NOTE]
Все подробности можно посмотреть на том же nvidia-tweaks

```bash
sudo bash -c 'mkdir -p /etc/environment.d && \
cat << _EOF_ >> /etc/environment.d/10-wayland.conf
CLUTTER_BACKEND=wayland
MOZ_DBUS_REMOTE=1
#_JAVA_AWT_WM_NONREPARENTING=1 #use only with on-reparenting window manager
ELECTRON_OZONE_PLATFORM_HINT=auto
_EOF_'
```
> [!NOTE]
> Логичней использовать окружения не сессий или пользователей, а сессии графических сред. Поэтому устанавливаем не в переменные оболочки или environment, а в переменные окружения wayland

>[!Warning]
>Не каждое окружение самостоятельно использует это расположение
>Для тайловых оконных менеджеров логичней использовать переменные внутри конфигов эти файлов

## Брандмауер:
**Установка:**
```bash
paru -S --needed ufw
```
**Стандартные настройки:**
```bash
sudo ufw default deny && \
sudo ufw allow from 192.168.0.0/24 && \
sudo ufw limit ssh
```
**Включение ufw:**
```bash
sudo ufw enable && \
sudo systemctl enable --now ufw
```

## Дополнительная безопасность:
### [ClamAV](/Программы%20и%20игры/ClamAV.md)

### [Timeshift](/Программы%20и%20игры/Timeshift.md)

### [UsbGuard](/Программы%20и%20игры/Usbguard.md)

### [Антивирус maldet](/Программы%20и%20игры/Антивирус%20maldet.md)

### Разрешения для важных файлов:
```bash
sudo chmod 600 /etc/cron.deny &&\
sudo chmod 644 /etc/group &&\
sudo chmod 644 /etc/group- &&\
sudo chmod 644 /etc/issue &&\
sudo chmod 644 /etc/passwd &&\
sudo chmod 644 /etc/passwd- &&\
sudo chmod 600 /etc/ssh/sshd_config &&\
sudo chmod 700 /root/.ssh &&\
sudo chmod 700 /etc/cron.d &&\
sudo chmod 700 /etc/cron.daily &&\
sudo chmod 700 /etc/cron.hourly &&\
sudo chmod 700 /etc/cron.weekly &&\
sudo chmod 700 /etc/cron.monthly &&\
sudo chmod 600 /etc/cron.deny &&\
sudo chmod 644 /etc/group &&\
sudo chmod 644 /etc/group- &&\
sudo chmod 644 /etc/issue &&\
sudo chmod 644 /etc/passwd &&\
sudo chmod 644 /etc/passwd- &&\
sudo chmod 600 /etc/ssh/sshd_config &&\
sudo chmod 700 /root/.ssh &&\
sudo chmod 700 /etc/cron.d &&\
sudo chmod 700 /etc/cron.daily &&\
sudo chmod 700 /etc/cron.hourly &&\
sudo chmod 700 /etc/cron.weekly &&\
sudo chmod 700 /etc/cron.monthly
```
### pam_pwquality
pam_pwquality предоставляет защиту от перебора по словарю и помогает настроить политики паролей для всей системы.
Добавление политики:
```bash
sudo bash -c 'cat << _EOF_ > "/etc/pam.d/passwd"
#%PAM-1.0
password required pam_pwquality.so retry=2 minlen=8 difok=6 dcredit=-1 ucredit=-1 lcredit=-1 [badwords=myservice mydomain] enforce_for_root
password required pam_unix.so use_authtok sha512 shadow
_EOF_'
```
>[!NOTE]
>При следующем изменении пароля, нужно будет создать пароль со следующими условиями:
>- запрашивать пароль 2 дополнительных раза в случае ошибки (параметр retry);
>- длина пароля не менее 8 символов (параметр minlen);
>- новый пароль должен отличаться от старого не менее чем шестью символами (параметр difok);
>- не менее 1 цифры (параметр dcredit);
>- не менее 1 буквы в верхнем регистре (параметр ucredit);
>- не менее 1 буквы в нижнем регистре (параметр lcredit);
>- не менее 1 другого символа (параметр ocredit);
>- не содержит слов "myservice" и "mydomain";
>- работает в том числе и для пользователя root.
>Подробнее тут: https://wiki.archlinux.org/title/Security_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)#%D0%A2%D1%80%D0%B5%D0%B1%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F_%D0%BA_%D0%BF%D0%B0%D1%80%D0%BE%D0%BB%D1%8E_%D1%81_pam_pwquality

## Настройка графического окружения:
>[!NOTE]
>Продолжаем установку DE

### [KDE](/Среды%20рабочего%20стола/KDE.md#kde-настройка-после-перезагрузки)

### [Gnome](/Среды%20рабочего%20стола/Gnome.md#Gnome-настройка-после-перезагрузки)

### En и Ру словари для aspell и huspell checkers:

```bash
sudo pacman -S aspell-ru hunspell-ru aspell-en hunspell-en_US
```

>[!NOTE}
>Это словари, которые необходимы системе для подчёркивания неправильных слов

## Далее будут личные настройки программы, которые можно установить уже после:

### [Flatpak](/Программы%20и%20игры/Flatpak.md)

### [ZSH](/Программы%20и%20игры/ZSH.md)

### [Neovim](/Программы%20и%20игры/Neovim.md)

### [Bottles и настройка программ](/Программы%20и%20игры/Bottles%20и%20настройка%20программ.md)

### [OpenRGB](/Программы%20и%20игры/OpenRGB.md)
### [Elder Scrools Online](/Программы%20и%20игры/Elder%20Scrools%20Online.md)


### Отключаем ssh, если не нужен:
```bash
sudo systemctl disable sshd.service
```

### Готово!
