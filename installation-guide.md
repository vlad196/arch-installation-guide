# Конфигурация для моей системы
# Моя система:
#### Видеокарта: nvidia
#### Процессор: amd
#### Диски:
#### (3 физических диска)
#### ||└───nvme0n1:
#### ||    | └──nvme0n1p1 -- efi 2gb
#### ||    └────nvme0n1p2 -- LUKS decrypt
#### ||          LUKS:system 
#### ||          | └───systemvg-swap 16 gb
#### ||          └─────systemvg-root 935 gb
#### |└─────sda -- windows 
#### └──────sdb -- дополнительный диск с ссылками папок в него

# Система Arch, особенности:
##### - Раскомментированы все тестовые и 32 битные репозитории пакетов
##### - Добавлено LQX ядро
#### - Корневой раздел и swap на одном зашифрованном диске LUKS
#### - Корневой раздел и swap это LVM тома
#### - Загрузка ядер, модулей и т.п. происходить через UKI из efi раздела
#### - Установлены твики для драйверов nvidia
#### - Настроен и включён Secure boot
#### - Ключ расшифровки раздела с LUKS прописан в tpm
#### - Настроен PLYMOUTH
#### - Настроены IPTABLES
#### - Включен Trim
#### - AppArmor
#### - Timeshift

## Обозначения:
### Машина-клиент -- машина, на которую устанавливают систему
### Машина-настройщик -- машина, через которую устанавливают систему по ssh
## Переменные:
>[!Info]
>Чтобы в последующем не путаться, я некоторые свои данные (Такие, как uuid дисков) переведу в переменные. 
>В принципе, можно на этом этапе передать переменные, чтобы потом не подставлять в тексте

>export NVME0N1P1 = A1D9-7475
>export NVME0N1P2 = 9b972571-bffd-4e65-8ae4-db9537b4c26a

## Внимание! Следите за данными, которые тут есть! Например диски могут отличаться, а UUID точно нужно подставлять свои. 
## Кроме того, следите за железом! Многое может не подойти к той или иной конфигурации ПК

# Часть 0. Подготовка
**1) Если решили заново настроить и нет уже готовых ключей для подписи, то убеждаемся, что Secure-boot сброшен и ключи пусты (Не вернувшиеся в дефолтное состояние, а именно пустые)**
**2) Проверяем, что для не подписанного arch iso был отключён Secure-boot**
**3) Проверяем, что TPM включён**

# Часть 1. Подготовка диска:
## Подключение по SSH:
#### Установка через ssh позволяет сразу копировать готовые команды
**На машине-клиент устанавливаем пароль:**
```bash
passwd
```
**На машине-клиент узнаём ip его локальный ip(Будет примерно 192.168.1.111):**
```bash
ip -br a
```
**Далее на машине-настройщике подключаемся по ssh к машине-клиенту**
```bash
ssh root@<ip машины-клиента, который узнали выше>
```
**Подгружаем локали (переключение будет по shift+ctrl)**
```bash
loadkeys ruwin_alt_sh-UTF-8
```

## Удаление данных с диска:
```bash
wipefs -a /dev/nvme0n1
```

## Создание разметки и разделов:
**Размечаем диск в gpt:**
```bash
parted /dev/nvme0n1 mklabel gpt
```
**Создаём 2 раздела:**
```bash
parted /dev/nvme0n1 mkpart "EFI system partition" fat32 2048s 2048MiB
parted /dev/nvme0n1 mkpart "primary partition" 2048MiB 100%
```
>[!Info]
>Очень важно, следите за размером секторов на диске! От этого зависит скорость доступа к диску. Если при создании раздела не соблюсти кратность сектора, то контроллер будет чаще обращаться к ячейкам, а значит увеличится и время доступа к данным. В данном случае, минимальный разрешённый сектор у меня 2048s(секторов).
>Дополнительно тут: https://wiki.archlinux.org/title/Parted_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)#%D0%92%D1%8B%D1%80%D0%B0%D0%B2%D0%BD%D0%B8%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5

>[!Info]
>Ещё интересная вещь, это указание размера разделов в процентах. Она очень удобна, когда нужно указать раздел от начала 0% или до конца 100%

**Назначаем флаги для разделов (необязательно, но пусть будет):**
```bash
parted /dev/nvme0n1 set 1 esp on
parted /dev/nvme0n1 set 2 LVM on
```
**Форматирование раздела под efi:**
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```
## LUKS шифрование раздела:
LUKS шифрование даёт нам раздел, который полностью закрыт для просмотра, делая из раздела, по сути один большой файл. Его структура похожа на структуру обычного диска с разметкой диска, только вместо разметки диска, у нас заголовок LUKS и ещё есть отдельный раздел для хранения ключей, за счёт чего в LUKS  можно удобно менять пароли (в отличии от того же dm-decrypt, где ключ выбирается один на все файлы)
Подробнее о моём варианте шифрования можно почитать тут:
https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
**Проверяем модули на работоспособность:**
```bash
modprobe dm-crypt
modprobe dm-mod
```
**Шифруем диск в SHA 512:**
```bash
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p2
```
**Открываем зашифрованный диск:**
```bash
cryptsetup luksOpen /dev/nvme0n1p2 LVM_part
```
**Создаём физический общий том:**
```bash
pvcreate /dev/mapper/LVM_part
```
**Создаём группу томов systemvg:**
```bash
vgcreate systemvg /dev/mapper/LVM_part
```
**Создаём логический топ для swap на 16 gb в группе systemvg:**
```bash
lvcreate -L16G -n swap systemvg
```
**Создаём логический том для корневого раздела root на остальное место в группе systemvg:**
```bash
lvcreate -l 100%FREE -n root systemvg
```
>[!Info]
>Очень удобно создавать последний диск параметрической переменной 100%FREE, которая создаёт логический том из оставшегося места
## Создание файловых систем в томах:
**Создание файловой системы swap:** 
```bash
mkswap -L swap /dev/mapper/systemvg-swap
```
**Создание файловой системы f2fs:**
```bash
mkfs.f2fs -f -O extra_attr,inode_checksum,sb_checksum,compression  /dev/mapper/systemvg-root
```
>[!Info]
>Для подробностей нужно идти на Archwiki, но тут, как минимум включена полезная компрессия
>Подробнее тут: https://wiki.archlinux.org/title/F2FS#Compression
>
## Монтирование разделов:
**Монтирование корневого раздела:**
```bash
mount -o compress_algorithm=zstd:6,compress_chksum,gc_merge,lazytime /dev/mapper/systemvg-root /mnt
```
>[!Info]
>Раздел смонтирован с рекомендуемыми Archwiki параметрами
>Подбробнее тут: https://wiki.archlinux.org/title/F2FS#Recommended_mount_options 

**Монтирование swap:**
```bash
swapon /dev/mapper/systemvg-swap
```
**Монтирование efi раздела:**
```bash
mount --mkdir /dev/nvme0n1p1 /mnt/efi
```
**Мой дополнительный диск:**
```bash
mount --mkdir /dev/sdb /mnt/mnt/sdb
```
# Часть 2. Начало настройки:
## Преднастройка:

**Подбор зеркал:**
Зачастую, стандартный подбор зеркал является неоптимальным. Поэтому с помощью reflector зеркала отсортируем по скорости и типу и сохраним их
```bash
reflector --verbose -l 5 -p https --sort rate --save /etc/pacman.d/mirrorlist
```
**Установка базовых пакетов:**
```bash
pacstrap -K /mnt base base-devel git nano vifm
```

**Генерирование FSTAB:**
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```
**Переход в CHROOT:**
```bash
arch-chroot /mnt
```
### Установка времени:
**Установка своего часового пояса:**
```bash
ln -sf /usr/share/zoneinfo/Europe/Kaliningrad /etc/localtime
```
>[!Info]
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
**Установка языка по умолчанию:**
```bash
echo LANG=ru_RU.UTF-8 >> /etc/locale.conf
```
**Установка шрифтов**
```bash
cat <<- _EOF_ > /etc/vconsole.conf
	KEYMAP=ruwin_alt_sh-UTF-8
	FONT=cyr-sun16
_EOF_
```
>[!Info]
>ruwin_alt_sh-UTF-8 раскладка, в отличии от раскладки, предлагаемой в "Installation guide" даёт возможность в режиме терминала менять раскладку через alt + shift
#### Создание хоста:
**Создание имени хоста:**
```bash
echo "arch" >> /etc/hostname
```
**Создание внутренней сети хоста:**
```bash
cat << _EOF_ >> /etc/hosts
127.0.0.1		localhost
::1			localhost
127.0.1.1		arch.localdomain	arch
_EOF_
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
**Для мнимой производительности корректируем флаги GCC:**
```bash
sed -i 's/-march=x86-64/-march=native/' /etc/makepkg.conf && \
sed -i 's/-mtune=generic/-mtune=native/' /etc/makepkg.conf && \
sed -i 's/#RUSTFLAGS="-C opt-level=2"/RUSTFLAGS="-C opt-level=3"/' /etc/makepkg.conf && \
sed -i 's/#MAKEFLAGS="-j2"/MAKEFLAGS="-j$(nproc) -l$(nproc)"/' /etc/makepkg.conf && \
sed -i 's/\!lto/lto/g' /etc/makepkg.conf
```
>[!Info]
>Все эти флаги взяты отсюда:
>https://ventureo.codeberg.page/v2022.07.01/source/generic-system-acceleration.html#makepkg-conf

### PARU:
Отличительной особенностью PARU является одновременно и то что он написан на Rust и то что он позволяет достаточно удобно работать с PKGBUILD 
**Устанавливаем его:**
```bash
sudo -u vlad git clone https://aur.archlinux.org/paru.git /home/vlad/bin/paru
cd /home/vlad/bin/paru/
sudo -u vlad makepkg -si
cd /
```

**Добавляем возможность редактирования в paru:**
```bash
sed '/\[bin\]/s/^#//' -i /etc/paru.conf && \
sed '/FileManager/s/^#//' -i /etc/paru.conf
```
### Добавление своего стека репозиториев:
**Делаем копию pacman.conf:**
```bash
cp /etc/pacman.conf /etc/pacman.conf.backup
```
**В /etc/pacman.conf раcкомментируем тестовые репозитории:**
```bash
sed '/Color/s/^#//' -i /etc/pacman.conf && \
sed '/\[core-testing\]/{s/^#//;n;s/^#//;}' -i /etc/pacman.conf && \
sed '/\[extra-testing\]/{s/^#//;n;s/^#//;}' -i /etc/pacman.conf && \
sed '/\[multilib-testing\]/{s/^#//;n;s/^#//;}' -i /etc/pacman.conf && \
sed '/\[multilib\]/{s/^#//;n;s/^#//;}' -i /etc/pacman.conf && \
sed '/\[core-testing\]/i\
\
# Default repositories\
' -i /etc/pacman.conf
```
**Раcкомментируем параллельные загрузки:**
```bash
sed '/ParallelDownloads =/s/^#//' -i /etc/pacman.conf
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
ALHP репозиторий (Репозиторий скомпилированный под 86-64-v3 архитектуру)
```bash
sed '/# Default repositories/i\
\# ALHP\
\[core-x86-64-v3\]\
Include = /etc/pacman.d/alhp-mirrorlist\
\
\[extra-x86-64-v3\]\
Include = /etc/pacman.d/alhp-mirrorlist\
' -i /etc/pacman.conf
```
llvm-svn (Репозиторий с последними скомилированными бинарниками llvm)
```bash
sed '/# Default repositories/i\
\# PKGBUILD for the LLVM\
\[llvm-svn\]\
\#SigLevel = Never\
\#Server = https://repos.uni-plovdiv.net/archlinux/\$repo/\$arch\
' -i /etc/pacman.conf
```
mesa-git (Репозиторий с последними скомилированными бинарниками mesa)
```bash
sed '/# Default repositories/i\
\[mesa-git\]\
\SigLevel = Never\
\Server = https://pkgbuild.com/~lcarlier/\$repo\/\$arch\
' -i /etc/pacman.conf
```
beta-kde (Репозиторий с beta версией KDE)
```bash
sed '/# Default repositories/i\
\# Beta-kde\
\[kde-unstable\]\
\Include = /etc/pacman.d/mirrorlist\
' -i /etc/pacman.conf
```

>[!Info]
>Репозитории написаны в порядке моего приоритета. С каждым новым вводом, следующий репозиторий будет записываться в конец списка

>[!Info]
>Следите за очерёдностью репозиториев. Репозитории расположенные вверху имеют приоритет перед нижними

### Обновление ключей и репозиториев:
**Инициализация:**
```bash
pacman-key --init
```
**Получить ключи из репозитория:**
```bash
pacman-key --populate archlinux
```
**Проверить текущие ключи на актуальность:**
```bash
pacman-key --refresh-keys
```
**Обновляем список пакетов:**
```bash
sudo -u vlad paru -Syu
```
### Добавление необходимых ключей:
**Добавляем ключ LQX сервера:**
```bash
pacman-key --keyserver hkps://keyserver.ubuntu.com --recv-keys 9AE4078033F8024D && pacman-key --lsign-key 9AE4078033F8024D
```
### Добавление пакетов: 
**Скачиваем необходимые пакеты. Ядро и LQX ядро, микрокод, f2fs пакеты, Nvidia драйвера, менеджер сети, менеджер efiboot, lvm2 и дополнительные шрифты:**
```bash
sudo -u vlad paru -S --needed linux-lts linux-lts-headers linux-firmware f2fs-tools amd-ucode linux-lqx linux-lqx-headers \
 nvidia-beta-dkms nvidia-utils-beta lib32-nvidia-utils-beta vulkan-icd-loader lib32-vulkan-icd-loader nvidia-settings-beta \
 efibootmgr networkmanager man nano lvm2 reflector pipewire pipewire-alsa noto-fonts-cjk ttf-hannom \
 ananicy-cpp wget
```
>[!linux-lqx ядро]
>Так как я хочу самостоятельно компилировать ядро, я добавил в paru возможность редактировать PKGBUILD.
>Тогда при использовании репозитория AUR скачивается версия ядра, которую нужно ещё скомпилировать.
>Когда paru дойдёт до linux-lqx, нам предложат изменить PKGBUILD. Меняем его и добавляем к _menunconfig= символ **y** 
>( Будет **_menunconfig=y**)
>Далее выбираем необходимые флаги. В моём случае:
> - Выбираем платформу zen2
> - Отключаем для видеокарты i2c (Видеокарта не поддерживает его)
> - Отключаем Kernel Debugging
>И продолжаем установку F9
>TODO: точно указать расположение этих флагов
>И продолжаем установку



> [!INFO]
>Для того, чтобы mkinitcpio не делал предупреждений о отсутствующих модулях, качаем
>этот пакет(Но это не обязательно т.к. оно ни на что не влияет):
>https://wiki.archlinux.org/title/Mkinitcpio#Possibly_missing_firmware_for_module_XXXX

## Первоначальная настройка:
### Включение сети:
**Включаем юнит менеджера сети:**
```bash
systemctl enable NetworkManager.service
```
**Включаем юнит ananicy-cpp:**
```bash
systemctl enable ananicy-cpp
```
### Редактирование MKINITCPIO:
**Делаем backup mkinitcpio.conf:**
```bash
cp /etc/mkinitcpio.conf /etc/mkinitcpio.conf.backup 
```
**Добавляем модули драйверов nvidia и crc32c:**
```bash
sed -e 's/\(MODULES=(\)/\1crc32c libcrc32c nvidia nvidia_modeset nvidia_uvm nvidia_drm/' -i /etc/mkinitcpio.conf
```
>[!Info]
>crc32c и libcrc32c это модули были взяты из Manjaro. 
>По сути является структурой для создания пар ключ-значение, придуманная google и нужна для проверки контрольной суммы файлов

**Редактировать hook mkinitcpio и включаем туда после block хуки lvm2 и encrypt:**
```bash
sed -i "/^HOOKS=/ s/\(block\)\(.*\)$/\1 encrypt lvm2\2/" /etc/mkinitcpio.conf
```
**Иногда ядро ругается на отсутствие KDFONTOP.  Это, скорее всего из-за нарушения очереди в HOOKS с plymouth:**
```bash
sed -e 's/\(BINARIES=(\)/\1setfont/' -i /etc/mkinitcpio.conf
```
### Установка HOOKS:
**Если папка не создалась, то создать:**
```bash
mkdir /etc/pacman.d/hooks
```
**Hook для Пересборки модулей ядра с драйверами nvidia при обновлении ядра**
```bash
cat << _EOF_ > /etc/pacman.d/hooks/nvidia.hook
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms-beta
Target=linux-lts
Target=linux-lqx
# Измените "linux" в строках Target и Exec, если вы используете другое ядро
# Если есть дополнительное ядро, то добавьте новым Target

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case \$trg in linux-lts|linux-lqx) exit 0; esac; done; /usr/bin/mkinitcpio -P'
# Если есть дополнительное ядро, то добавьте его через | 
_EOF_
```
> [!INFO]
>Измените "linux" в строках Target и Exec, если вы используете другое ядро
>Если есть дополнительное ядро, то добавьте новым Target

**Hook для обновления микрокода:**
```bash
cat << _EOF_ > /etc/pacman.d/hooks/microcode_reload.hook
[Trigger]
Operation = Install
Operation = Upgrade
Operation = Remove
Type = File
Target = usr/lib/firmware/amd-ucode/*	

[Action]
Description = Applying CPU microcode updates...
When = PostTransaction
Depends = sh
Exec = /bin/sh -c 'echo 1 > /sys/devices/system/cpu/microcode/reload'
_EOF_
```



## NVIDIA TWEAKS
Множественные исправления, большую часть которых взял отсюда (https://github.com/ventureoo/nvidia-tweaks)
Исправляет следующее:
- потеря памяти в спящем режиме nvidia-beta-dkms
- добавляет поддержку PAT
- "потенциально" ускоряет производительность за счёт безопасности (Если выберите безопасность, то закомментируйте NVreg_InitializeSystemMemoryAllocations)
и т.п. 
Всё расписано в самом конфиге, либо же, можно посмотреть на самом гитхабе выше.

Закомментируемы некоторые опции, т.к. лично у меня они либо не нужны, либо не поддерживаются, либо вовсе влекут за собой проблемы. 
Закомментированы следующие:
- NVreg_EnableResizableBar т.к. моё устройство не поддерживает resizable bar
- NVreg_TemporaryFilePath т.к. не без него ждущий режим работает хорошо, а сама опция подразумевает лишнее использование циклов записи для устройства хранения
- NVreg_RegistryDwords т.к. вроде как необходим для некоторых ноутбуков для правильного отображения nvidia-settings и он мне не нужен
- NVreg_EnableGpuFirmware т.к. лично у меня этот параметр ломает выход из спящего режима видеокарты
**Добавляем в modprobe.d:**
```bash
cat << _EOF_ > /etc/modprobe.d/nvidia-tweaks.conf
options nvidia NVreg_PreserveVideoMemoryAllocations=1
#
# Allow to preserve memory allocations (Required to properly wake up from sleep mode)

#options nvidia NVreg_TemporaryFilePath=/var/tmp
#
# Msy help with suspend issue. Turn on if suspend not working.

options nvidia NVreg_UsePageAttributeTable=1
#
# NVreg_UsePageAttributeTable=1 (Default 0) - Activating the better
# memory management method (PAT). The PAT method creates a partition type table
# at a specific address mapped inside the register and utilizes the memory architecture
# and instruction set more efficiently and faster.
# If your system can support this feature, it should improve CPU performance.

options nvidia NVreg_InitializeSystemMemoryAllocations=0
#
# NVreg_InitializeSystemMemoryAllocations=0 (Default 1) - Disables
# clearing system memory allocation before using it for the GPU.
# Potentially improves performance, but at the cost of increased security risks.
# Write "options nvidia NVreg_InitializeSystemMemoryAllocations=1" in /etc/modprobe.d/nvidia.conf,
# if you want to return the default value.
# Note: It is possible to use more video memory (?)

options nvidia NVreg_EnableStreamMemOPs=1
#
# NVreg_EnableStreamMemOPs=1 (Default 0) - Activates the support for
# CUDA Stream Memory Operations in user-mode applications.

#options nvidia NVreg_RegistryDwords=__REGISTRYDWORDS
#
# Need for some notebooks, for correctly open nvidia-settings?

softdep nvidia post: nvidia-uvm
#
# Make a soft dependency for nvidia-uvm as adding the module loading to
# /usr/lib/modules-load.d/nvidia-uvm.conf for systemd consumption, makes the
# configuration file to be added to the initrd but not the module, throwing an
# error on plymouth about not being able to find the module.
# Ref: /usr/lib/dracut/modules.d/00systemd/module-setup.sh

# Even adding the module is not the correct thing, as we don't want it to be
# included in the initrd, so use this configuration file to specify the
# dependency.

options nvidia NVreg_EnableStreamMemOPs=1
options nvidia NVreg_DynamicPowerManagement=0x02
options nvidia NVreg_EnableS0ixPowerManagement=1
#
# Enable complete power management. From:
# file:///usr/share/doc/nvidia-driver/html/powermanagement.html

#options nvidia NVreg_EnableResizableBar=1
#
#https://www.nvidia.com/en-us/geforce/news/geforce-rtx-30-series-resizable-bar-support/
# Working just for RTX 3xxx series and CPU since amd ryzen 5xxx\ intel 10th gen
# So you can enable it, if you have it
#(also, you need enable resizable bar in the motherboard)

#options nvidia NVreg_EnableGpuFirmware=1
# (May crash suspend. Need to check)
# GSP (GPU System Processor) - this is a special chip which is present on NVIDIA video cards starting
# from Turing and above, which offloads GPU initialization and control tasks, which are usually
#performed on CPU. This should improve performance and reduce the load on the CPU.
#WARNING: I strongly suggest forcing the use of GSP firmware only on the most recent driver, as the first
#releases with its support may contain certain problems. Only starting from 530 you will have support
#for suspend and resume when using GSP firmware. This feature can also work badly on PRIME configurations,
#so please check dmesg logs for errors if you want to use this.

blacklist nouveau
#
# Nouveau must be blacklisted here as well beside from the initrd to avoid a
# delayed loading (for example on Optimus laptops where the Nvidia card is not
# driving the main display).

#blacklist i2c_nvidia_gpu
#
# If the video card does not have type-c, and support is included in the drivers, then you can mute it
_EOF_
```

**Включаем suspend от nvidia:**
```bash
systemctl enable nvidia-suspend
```
**Включаем hibernate от nvidia:**
```bash
systemctl enable nvidia-hibernate
```
> [!INFO]
>Можно и не включать, если не используете гибернацию. 
>Лично я включил только ради того, чтобы кнопка в пуске работала штатно, если когда-нибудь её нажму

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

### Установка UKI:
**Добавить опции основного ядра в cmdline:**
```bash
echo "options page_alloc.shuffle=1 cryptdevice=UUID=$NVME0N1P2:LVM_part:allow-discards root=/dev/systemvg/root rootflags=atgc rw nvidia_drm.modeset=1" >> \
/etc/kernel/cmdline
```
>[!Info]
>**page_alloc.shuffle=1** - Этот параметр рандомизирует свободные списки
> распределителя страниц. Улучшает производительность при работе с ОЗУ с очень быстрыми накопителями (NVMe, Optane). Подробнее [тут](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e900a918b0984ec8f2eb150b8477a47b75d17692).
>Доп. информация здесь: https://ventureo.codeberg.page/source/kernel-parameters.html
>**cryptdevice**= - опция, указывающая какой LUKS раздел открывать
>**:allow-discards** - параметр опции, разрешающий trim
>**root=** - опция, указывающая какой раздел грузить как root (в данном случае это логический диск)
>**rootflags=atgc** - опцию, которую я не расшифровал, но тут написано зачем оно:https://wiki.archlinux.org/title/F2FS#Remounting_impossible_with_some_options
>**rw** - разрешение на чтение запись раздела
>**nvidia_drm.modeset=1** - разрешение ранней загрузки видеоядра

>[!Info]
>В **cryptdevice=UUID=9b972571-bffd-4e65-8ae4-db9537b4c26a** должен быть UUID самого luks раздела. Тут можно легко запутаться и вставить не тот UUID. Если это случилось и система потом не грузиться, то просто через arch iso опять подключаетесь и меняем на нужный UUID

**Добавим опции для вспомогательного ядра (В итоге, он как запасной имеет минимальные для загрузки параметры, например будет в дальнейшем без plymouth параметров):**
```bash
echo "options page_alloc.shuffle=1 cryptdevice=UUID=$NVME0N1P2:LVM_part:allow-discards root=/dev/systemvg/root rootflags=atgc rw nvidia_drm.modeset=1" >> \
/etc/kernel/cmdline-base
```
**Для каждого ядра изменить пресеты( например, в linux.preset будет cmdline-base):**
```bash
cat << _EOF_ > /etc/mkinitcpio.d/linux-lts.preset
# mkinitcpio preset file for the 'linux' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-lts"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-lts.img"
default_uki="/efi/EFI/Linux/arch-linux-lts.efi"
default_options="--cmdline /etc/kernel/cmdline-base"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-lts-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-lts-fallback.efi"
fallback_options="-S autodetect"
_EOF_
```

```bash
cat << _EOF_ > /etc/mkinitcpio.d/linux-lqx.preset
# mkinitcpio preset file for the 'linux-lqx' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-lqx"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-lqx.img"
default_uki="/efi/EFI/Linux/arch-linux-lqx.efi"
default_options=""

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-lqx-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-lqx-fallback.efi"
fallback_options="-S autodetect"
_EOF_
```
**Пересобираем ядра уже в EFI:**
```bash
mkinitcpio -P
```
**Из-за ненадобности удаляем initramfs-linux из boot, т.к. отныне, это всё будет в efi файле:**
```bash
rm /boot/initramfs-linux*
```

### Подпись файлов efi созданными ключами:
**Подписываем linux efi приложения с зашитыми ядрами, модулями, конфигами и т.п.:**
```bash
sbctl sign -s /efi/EFI/Linux/arch-linux-lts.efi && \
sbctl sign -s /efi/EFI/Linux/arch-linux-lts-fallback.efi && \
sbctl sign -s /efi/EFI/Linux/arch-linux-lqx.efi && \
sbctl sign -s /efi/EFI/Linux/arch-linux-lqx-fallback.efi
```
> [!INFO]
> sbctl создаёт после этого hook, который будет постоянно их подписывать)

**Подписываем Windows файлы:**
```bash
sbctl sign -s /efi/EFI/Boot/bootx64.efi && \
sbctl sign -s /efi/EFI/Microsoft/Boot/bootmgfw.efi && \
sbctl sign -s /efi/EFI/Microsoft/Boot/bootmgr.efi && \
sbctl sign -s /efi/EFI/Microsoft/Boot/memtest.efi
```

## Загрузчики:

### Systemd-boot:
```bash
bootctl install
```
**Подписываем systemd-boot файл:**
```bash
sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
```
#### Конфигурируем systemd-boot:
**В папке loader создать и настроить:**
```bash
cat << _EOF_ > /efi/loader/loader.conf
default arch-linux-lqx.efi
timeout 10
console-mode max
editor 1
_EOF_
```
> [!INFO]
> Далее будет grub, который можно установить либо рядом, либо как замену

### GRUB
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
**Устанавливаем и генерируем grub:**
```bash
grub-install --efi-directory=/efi --boot-directory=/efi --bootloader-id=GRUB --modules="tpm" --disable-shim-lock --debug
grub-mkconfig -o /efi/grub/grub.cfg 
```
**Подписываем файлы:**
```bash
sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI && sudo sbctl sign -s /efi/EFI/GRUB/grubx64.efi 
```
#### Тема:
**distro-grub-themes\\asrock:**
```bash
cd /home/vlad/bin
wget https://github.com/AdisonCavani/distro-grub-themes/raw/master/themes/asrock.tar
mkdir ./asrock

tar -C /home/vlad/bin/asrock -xvf /home/vlad/bin/asrock.tar
cp -r /home/vlad/bin/asrock /efi/grub/themes/

sed '/GRUB_THEME/s/^#//' -i /etc/default/grub && \
sed -i 's|GRUB_THEME="/path/to/gfxtheme"|GRUB_THEME=/efi/grub/themes/asrock/theme.txt|' /etc/default/grub && \
sed -i 's|GRUB_GFXMODE=auto|GRUB_GFXMODE=1920x1080|' /etc/default/grub
```

## Без загрузчика:
> [!INFO]
> Т.к. мы используем UKI, можно сделать загрузку напрямую без загрузчика из UEFI
> Есть минусы:
> - Не будут работать такие вещи как fwupd
> - Не будет никаких дополнительных вариантов загрузки
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Arch Linux" --loader 'EFI\Linux\arch-linux-lqx.efi' --unicode
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
paru -S fwupd
```
**Подписываем:**
```bash
sbctl sign -s /efi/EFI/Linux/fwupd.efi
```
**Добавляем триггер для подписи fwupd при его обновлении:**
```bash
cat << _EOF_ >> /etc/pacman.d/hooks/sign-fwupd-secureboot.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/fwupd/efi/fwupdx64.efi

[Action]
When = PostTransaction
Exec = /usr/bin/sbctl sign-all
Depends = sbctl
_EOF_
```
## Постзагрузка:
### Plymouth:
**Установка:**
```bash
sudo -u vlad paru -S plymouth plymouth-theme-arch-bgrt
```
**В cmdline добавить:**
```bash
sed -i -e 's/$/ loglevel=3 quiet splash rd.udev.log_priority=3 vt.global_cursor_default=0/' /etc/kernel/cmdline
```
**В hook в mkinitcpio после udev вставить plymouth:**
```bash
sed -i "/^HOOKS=/ s/\(udev\)\(.*\)$/\1 plymouth\2/" /etc/mkinitcpio.conf
```
> [!INFO]
> У nvidia plymouth появляется довольно поздно и это нормально. 
> Всё из-за поздней загрузки kms

**Выбираем тему arch-bqrt:**
```
plymouth-set-default-theme -R arch-bgrt
```
### Настройка plymouth:
### Гибернация:
**В hook в mkinitcpio после lvm2 вставляем resume:**
```bash
sed -i "/^HOOKS=/ s/\(lvm2\)\(.*\)$/\1 resume\2/" /etc/mkinitcpio.conf
```
**В cmdline добавляем resume:**
sed -i -e 's/$/ resume=\/dev\/systemvg\/swap/' /etc/kernel/cmdline
## Начало настройки рабочего окружения:

### Связываем рабочие папки с дополнительным диском:
**Добавить ссылки Загрузок, документов И Т.П. с дополнительного диска на основную папку:**
```bash
ln -s /mnt/sdb/Изображения /home/vlad/ && \
ln -s /mnt/sdb/Видео /home/vlad/ && \
ln -s /mnt/sdb/Загрузки /home/vlad/ && \
ln -s /mnt/sdb/Документы /home/vlad/ && \
ln -s /mnt/sdb/Музыка /home/vlad/
```
> [!INFO]
> Только если Вам действительно нужно, чтобы эти папки были на другом диске

## Графические окружения:
> [!INFO]
> Желательно выбрать одно из окружений, т.к. при параллельном использовании одно окружение может влиять на другое

### KDE:
**Установка:**
```bash
sudo -u vlad paru -S --needed plasma-meta sddm-git plasma-wayland-session egl-wayland xorg-xwayland xorg-xlsclients qt5-wayland glfw-wayland kde-system-meta ark filelight kalk kate kcharselect kclock kdebugsettings kdf kdialog keditbookmarks keysmith kfind kgpg konsole krecorder ktimer kweather markdownpart sweeper yakuake  kbackup kteatime kdeconnect ktorrent ufw
```
**Включение экранного менеджера:**
```bash
systemctl enable sddm.service
```
#### Для плавного перехода plymouth, нужно прописать следующее:
**Для начала создаём папку, если её нет:**
```bash
mkdir /etc/systemd/system/display-manager.service.d
```
**Добавляем юнит:**
```bash
cat << _EOF_ > /etc/systemd/system/display-manager.service.d/plymouth.conf
[Unit]
Conflicts=plymouth-quit.service
After=plymouth-quit.service rc-local.service plymouth-start.service systemd-user-sessions.service
OnFailure=plymouth-quit.service

[Service]
ExecStartPre=-/usr/bin/plymouth deactivate
ExecStartPost=-/usr/bin/sleep 30
ExecStartPost=-/usr/bin/plymouth quit --retain-splash
_EOF_
```
**SDDM под wayland:**
```bash
cat << _EOF_ >/etc/sddm.conf.d/10-wayland.conf
[General]
DisplayServer=wayland
GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell
_EOF_
```

### Шрифты:
```bash
sudo -u vlad paru -S ttf-ubuntu-nerd ttf-spacemono ttf-meslo-nerd-font-powerlevel10k
```

### Trim:
#### Настройка Trim для lvm:
**Сделать бэкап /etc/lvm/lvm.conf:**
```bash
cp /etc/lvm/lvm.conf /etc/lvm/lvm.conf.backup
```
**В /etc/lvm/lvm.conf расскомментить issue_discards = 1:**
```bash
sed -i 's/# issue_discards = 0/issue_discards = 1/g' -i /etc/lvm/lvm.conf
```

#### Непрерывный Trim:
В F2FS по умолчанию включён f2fs, который ведёт себя как непрерывный со своими особенностями

#### Периодический Trim:
**В /etc/fstab добавить nodiscard:**
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```
**Включение, старт и вывод периодического Trim:**
```bash
systemctl enable fstrim.service && \
systemctl start fstrim.service && \
systemctl status fstrim.service
```
> [!INFO]
> Естественно, если стоит f2fs, то включать периодический Trim не стоит. У f2fs есть свой Trim 

### SSH
**Установка**:
```bash
sudo -u vlad paru -S openssh
```
%%
TODO: надо сюда добавить настройки по безопасности
%%
**Включаем юнит:**
```bash
systemctl enable sshd.service
```

### Перезагружаемся
> [!INFO]
Обязательно проверяем, что UKI файлы на 100% собраны и без ошибок
Так как в этап создания initramfs теперь подвязаны драйвера nvidia и подпись файлов для secure boot,
то теперь (а логичней сделать отдельный хук для этого) нужно чтобы хуки от nvidia и подписи срабатывали
Нужно в конце сгенерировать mkibitcpio и подписать ядра:

```bash
mkinitcpio -P ; sudo sbctl sign-all
```
**Выходим из chroot:**
```bash
exit
```
**Отмонтируем все диски:**
```bash
umount -a
```

**Перезагружаемся:**
```bash
reboot
```

## Включаем secure boot и TPMtrusted compute в UEFI

---
# Всё что ниже нужно делать уже после перезагрузки системы
___
# Часть 3. Продолжение настройки настройки:
## Повторное подключение по SSH:

**На машине-клиент снова узнаём ip его локальный ip(Будет примерно 192.168.1.111):**
```bash
ip -br a
```
**Далее на машине-настройщике подключаемся по ssh к машине-клиенту:**
```bash
ssh vlad@<ip машины-клиента, который узнали выше>
```
## Настройка LUKS ЧЕРЕЗ TPMtrusted compute
>[!Info]
>Помимо systemd-cryptenroll есть ещё Clevis, но он первый в итоге себя показал намного быстрее при загрузке
### Systemd-cryptenroll:
**Устанавливаем необходимые пакеты:**
```bash
paru -S --needed tpm2-tss
```
**Проверяем распознаёт ли linux tpm модуль(Если при загрузке появляется ошибка tpm, то ничего страшного):**
```bash
test -c /dev/tpm0 && echo OK || echo FAIL
```
> [!INFO]
**Если нет, то проверяем включён ли TPM модуль в материнке:**

**Создаём копию заголовка (Её лучше сразу на какую-то флешку перекинуть):**
```bash
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /mnt/sdb/header-nvme0n1p2.img
```
%%
!!! НАДО ПОСМОТРЕТЬ, КАК ВОССТАНАВЛИВАТЬ ЭТИ ЗАГОЛОВКИ, НА БУДУЩЕЕ
%%
**Привязываем luks к clevis и внедряем в tpm ключ:**
```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme0n1p2
```
**Проверяем всё ли записалось:**
```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```
**Добавляем /etc/crypttab.initramfs: **
```bash
echo "system /dev/nvme0n1p2 none timeout=180,tpm2-device=auto" > /etc/crypttab.initramfs
```
**Меняем в mkinitcpio.conf некоторые модули на модули systemd:**
```bash
sudo sed -i 's/udev/systemd/' /etc/mkinitcpio.conf && \
sudo sed -i 's/keymap consolefont/sd-vconsole/' /etc/mkinitcpio.conf && \
sudo sed -i 's/encrypt/sd-encrypt/' /etc/mkinitcpio.conf && \
sudo sed -i 's/resume//' /etc/mkinitcpio.conf
```
**Обновляем ядра и подписываем их:**
```bash
sudo mkinitcpio -P ; sudo sbctl sign-all
```

## Переменные для wayland

> [!INFO]
Все подробности можно посмотреть на том же nvidia-tweaks

```bash
mkdir ~/.config/environment.d
cat << _EOF_ >> ~/.config/environment.d/envvars.conf
# Wayland environment
SDL_VIDEODRIVER=wayland # Can break some native games
XDG_SESSION_TYPE=wayland
QT_QPA_PLATFORM=wayland
MOZ_ENABLE_WAYLAND=1
GBM_BACKEND=nvidia-drm
WLR_NO_HARDWARE_CURSORS=1 
KITTY_ENABLE_WAYLAND=1
#LIBVA_DRIVER_NAME,nvidia
__GLX_VENDOR_LIBRARY_NAME=nvidia
_EOF_
```
> [!INFO]
> Логичней использовать окружения не сессий или пользователей, а сессии графических сред. Поэтому устанавливаем не в переменные оболочки или environment, а в переменные окружения wayland

## Брандмауер:
**Установка:**
```bash
paru -S ufw
```
**Стандартные настройки:**
```bash
sudo ufw default deny && \
sudo ufw allow from 192.168.0.0/24 && \
sudo ufw allow Deluge && \
sudo ufw limit ssh
```
**Включение ufw:**
```bash
sudo ufw enable && \
sudo systemctl enable --now ufw
```

## Настройка графического окружения:
>[!Info]
>Выбираем именно то графическое окружение, что установили
### KDE настройка:
**Настройка ufw для kdeconnect:**
```bash
sudo ufw allow 1714:1764/udp && \
sudo ufw allow 1714:1764/tcp
```
**Создаём конфиг файл sddm:**
```bash
sudo bash -c 'cat << _EOF_ > /etc/sddm.conf.d/kde_settings.conf
[Autologin]
Relogin=false
Session=
User=

[General]
HaltCommand=/usr/bin/systemctl poweroff
RebootCommand=/usr/bin/systemctl reboot


[Theme]
Current=breeze

[Users]
MaximumUid=60513
MinimumUid=1000
_EOF_'
```

**Включить NUMLOCK в /etc/sddm.conf.d/kde_settings.conf:**
```bash
sudo sed -i "/\[General\]/{ 
n
n
a\\
Numlock=on \\n
}" /etc/sddm.conf.d/kde_settings.conf
```

**Установить русскую раскладку:**
```bash
cat << _EOF_ >> ~/.config/kxkbrc

[Layout]
DisplayNames=,
LayoutList=us,ru
Use=true
VariantList=,typo
_EOF_
```

**Добавляем сочетание клавиш смены языка как в gnome:**
Увы, пока только через Gui настройки

**Включить Numlock в самой kde:**
```bash
sed -i "/\[\$Version\]/{ 
n
n
a\\
\[Keyboard\]\\
NumLock=0 \n
}" $HOME/.config/kcminputrc
```

**Добавление аватара:**
```bash
sudo curl -o /var/lib/AccountsService/icons/vlad https://besplatnye-programmy.com/uploads/posts/2021-04/1617720980_arch-linux.png
```

**Установка packagekit и нужных себе программ:**
```bash
paru -S packagekit-qt5 ktorrent
```
## Дополнительная безопасность:
### AppArmor
>[!Info]
Прежде всего нужно убедиться, что ядро поддерживает apparmor

**Т.к. установщик apparmor-git ругается на отсутствие флагов в ядре, то сначала прописываем их:**
```bash
sudo sed -i -e 's/$/ lsm=landlock,lockdown,yama,integrity,apparmor,bpf audit=1/' /etc/kernel/cmdline
```
**Обновляем и подписываем ядро:**
```bash
sudo mkinitcpio -P ; sudo sbctl sign-all
```
**Перезагружаемся для того, чтобы применились новые параметры для ядра:**
```bash
sudo reboot
```
**Устанавливаем сам apparmor (audit тоже качает ):**
```bash
paru -S apparmor-git
```
**Запускаем юнит apparmor:**
```bash
systemctl enable --now apparmor
```

#### Для работы уведомлений от Apparmor нужно:
**Создаём группу аудита:**
```bash
sudo groupadd -r audit
```
**Создаём добавляем юзера в группу аудита:**
```bash
sudo gpasswd -a vlad audit
```
**Добавляем группу аудита в конфига аудита:**
```bash
sudo sed -i '3 i log_group = audit' /etc/audit/auditd.conf
```
**Создаём папку автостарт, если нет:**
```bash
mkdir ~/.config/autostart
```
**Добавляем .desktop файл для уведомления:**
```bash
cat << _EOF_ >> ~/.config/autostart/apparmor-notify.desktop
[Desktop Entry]
Type=Application
Name=AppArmor Notify
Comment=Receive on screen notifications of AppArmor denials
TryExec=aa-notify
Exec=aa-notify -p -s 1 -w 60 -f /var/log/audit/audit.log
StartupNotify=false
NoDisplay=true
_EOF_
```
**Запускаем аудита:**
```bash
systemctl enable --now auditd
```
**Создаём файл syslog, если его нет:**
```
sudo touch /var/log/syslog
```
**Перезагружаемся:**
```bash
sudo reboot
```
**Проверяем, работает ли aa-notify:**
```bash
pgrep -ax aa-notify
```
#### Настройка для кеширования профилей:
**Смотрим время загрузки профилей:**
```bash
systemd-analyze blame | grep apparmor
```
**Сниманием комментарий с write-cahe:**
```bash
sudo sed '/write-cache/s/^#//' -i /etc/apparmor/parser.conf
```
**Перезагружаемся:**
```bash
sudo reboot
```
**Смотрим время загрузки профилей, теперь должно быть меньше:**
```bash
systemd-analyze blame | grep apparmor
```

### Timeshift
**Установка**:
```bash
paru -S timeshift
```
>[!Далее настройка только через GUI:]
>1. Выбрать Rcync, т.к. у нас не BTRFS (Но если будет btrfs, то естественно его, если хотим использовать снапшоты)
>2. Выбрать диск где будет бэкап
>3. Выбрать периодичность


### UsbGuard
**Установка:**
```bash
paru -S usbguard-git
```

**Разрешаем все подключенные устройства:**
```bash
sudo bash -c "usbguard generate-policy > /etc/usbguard/rules.conf"
```

**Включаем юнита:**
```bash
sudo systemctl enable --now usbguard-dbus
```

**Создаём polkit для группы wheel в гноме:**
```bash
sudo zsh -c 'cat << _EOF_ >> /etc/polkit-1/rules.d/70-allow-usbguard.rules
// Allow users in wheel group to communicate with USBGuard
polkit.addRule(function(action, subject) {
    if ((action.id == "org.usbguard.Policy1.listRules" ||
         action.id == "org.usbguard.Policy1.appendRule" ||
         action.id == "org.usbguard.Policy1.removeRule" ||
         action.id == "org.usbguard.Devices1.applyDevicePolicy" ||
         action.id == "org.usbguard.Devices1.listDevices" ||
         action.id == "org.usbguard1.getParameter" ||
         action.id == "org.usbguard1.setParameter") &&
        subject.active == true && subject.local == true &&
        subject.isInGroup("wheel")) {
            return polkit.Result.YES;
    }
});
_EOF_'
```

**Включаем уведомления в гноме:**
```bash
gsettings set org.gnome.desktop.privacy usb-protection true
```
>[!Info]
>За дополнительной информацией: https://wiki.archlinux.org/title/USBGuard#Usage

### NTP
**Установка:**
```bash
paru -S ntp
```
**Включение юнита ntp:**
```bash
sudo systemctl enable --now ntpd
```
>[!Info]
>За дополнительной информацией: https://wiki.archlinux.org/title/Network_Time_Protocol_daemon_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)

### Антивирус maldet
**Установка:**
```bash
paru -S maldet
```
**Включение юнита maldet-update-signatures.timer:**
```bash
systemctl enable --now maldet-update-signatures.timer
```
**Обновление maldet-update-signatures.timer:**
```bash
systemctl start maldet-update-signatures.service
```
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
password required pam_pwquality.so retry=2 minlen=10 difok=6 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1 [badwords=myservice mydomain] enforce_for_root
password required pam_unix.so use_authtok sha512 shadow
_EOF_'
```
>[!Info]
>При следующем изменении пароля, нужно будет создать пароль со следующими условиями:
>- запрашивать пароль 2 дополнительных раза в случае ошибки (параметр retry);
>- длина пароля не менее 10 символов (параметр minlen);
>- новый пароль должен отличаться от старого не менее чем шестью символами (параметр difok);
>- не менее 1 цифры (параметр dcredit);
>- не менее 1 буквы в верхнем регистре (параметр ucredit);
>- не менее 1 буквы в нижнем регистре (параметр lcredit);
>- не менее 1 другого символа (параметр ocredit);
>- не содержит слов "myservice" и "mydomain";
>- работает в том числе и для пользователя root.
>Подробнее тут: https://wiki.archlinux.org/title/Security_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)#%D0%A2%D1%80%D0%B5%D0%B1%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D1%8F_%D0%BA_%D0%BF%D0%B0%D1%80%D0%BE%D0%BB%D1%8E_%D1%81_pam_pwquality


## Далее будут личные настройки программы, которые можно установить уже после:

### ZSH
**Установка:**
```bash
paru -S zsh zsh-completions awesome-terminal-fonts
```
**Установка oh-my-zsh:**
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
**Установить тему для zsh:**
```bash
paru -S --noconfirm zsh-theme-powerlevel10k-git
```
```bash
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
```
```
zsh
```
**Сменяем дефолтный Shell на zsh:**
```bash
chsh -s /bin/zsh
```
### Flatpak
**Установка:**
```bash
paru -S flatpak
```
**Устанавливаем мои приложения:**
```bash
flatpak install flathub md.obsidian.Obsidian \
flathub org.telegram.desktop \
flatpak install flathub org.mozilla.firefox
```
**Для Obsidian добавляем переменную для wayland:**
```bash
cat << _EOF_ >> ~/.zshrc

# Wayland environment
flatpak override --user --env=OBSIDIAN_USE_WAYLAND=1 md.obsidian.Obsidian
_EOF_
```
### Neovim
**Установка:**
```bash
paru -S neovim npm ripgrep lazygit
```
### Astronvim
**Создаём папку для AstroNvim:**
```bash
mkdir -p ~/.config/nvim/AstroNvim
```
**Установка:**
```bash
git clone --depth 1 https://github.com/AstroNvim/AstroNvim ~/.config/AstroNvim
```

**Nvim environment:**
```bash
 cat << _EOF_ >> ~/.zshrc

## NeoVim environment
alias nvim-astro="NVIM_APPNAME=AstroNvim nvim"
_EOF_
```
>[!Info]
>Переменная NVIM_APPNAME даёт возможность пользоваться разные конфигурациями, в зависимости от нужды. Достаточно записать файлы конфигурации в дополнительную папку, которую потом можно передать этой переменной.
### Yandex-disk
```bash
mkdir -p /mnt/sdb/Yandex.Disk
```
**Делаем ссылки на основные папки:**
```bash
ln -s /mnt/sdb/Документы /mnt/sdb/Yandex.Disk/Компьютер\ My_computer/ && \
ln -s /mnt/sdb/Изображения /mnt/sdb/Yandex.Disk/Компьютер\ My_computer/
```
**Установка:**
```bash
paru -S yandex-disk
```
**Запускаем установку и следуем инструкции:**
```bash
yandex-disk setup
```
Конфигурации:
proxy: no
/mnt/sdb/Yandex.Disk

## Программы:
### Bottles
**Установка:**
```bash
paru -S bottles
```
#### Sketchup:
Зависимости:
Библиотеки vsredist2019 dotnet48 allfonts

## Игры:
### Steam
**Установка:**
```bash
paru -S steam
```

### Elder Scrools Online:
Во время установки будет проблема с местом на диске
Нужно в аргументы игры в Steam добавить:
```bash
PROTON_SET_GAME_DRIVE=1 %command%
```
Качаем из Steam 
>[!Info]
Если при запуске только маленькое окошко, то это трей, можно оттуда вызвать лаунчер)

### Minion:
**Устанавливаем:**
```bash
paru -S miniongg
```
В режиме x11 выбираем нужные аддоны
#### Плагин: TTC 
https://esoui.com/downloads/info3249-LinuxTamrielTradeCenter.html 
Вместо Downloads заменяем на любую другую удобную папку
harvest map data запускаем через sh

#### Отключаем ssh, если не нужен:
```bash
sudo systemctl disable sshd.service
```

### Готово!

