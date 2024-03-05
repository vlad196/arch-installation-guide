## NVIDIA:
**Добавляем модули драйверов nvidia:**
```bash
sed -e 's/\(MODULES=(\)/\1nvidia nvidia_modeset nvidia_uvm nvidia_drm/' -i /etc/mkinitcpio.conf.d/mkinitcpio.conf 
```
#### Установка HOOK для драйверов:
**Hook для Пересборки модулей ядра с драйверами nvidia при обновлении ядра**
```bash
cat << _EOF_ > /etc/pacman.d/hooks/nvidia.hook
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-dkms*

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
Depends=sbctl
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case \$trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P; /usr/bin/sbctl sign-all'
_EOF_
```
>[!NOTE]
К сожалению, с AUR скриптами хуки не работают. Заработает если будет обычный пакет nvidia-dkms

**Добавление репозиториев mesa-git (Репозиторий с последними скомилированными бинарниками mesa)**
```bash
sed '/# Default repositories/i\
\[mesa-git\]\
\SigLevel = Never\
\Server = https://pkgbuild.com/~lcarlier/\$repo\/\$arch\
' -i /etc/pacman.conf
```
**Устанавливаем пакеты:**
```bash
sudo -u vlad paru -Sy --needed nvidia-beta-dkms lib32-nvidia-utils mesa-git lib32-mesa-git
```
>[!Note]
>nvidia-beta-dkms может ругаться на зависимости, если они тоже были явно установлены, поэтому вместо nvidia-beta-dkms nvidia-utils-beta и nvidia-settings-beta можно просто nvidia-beta-dkms
### NVIDIA TWEAKS
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

#options nvidia NVreg_RegistryDwords="PowerMizerDefaultAC=0x1"
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

options nvidia NVreg_EnableGpuFirmware=1
# (May crash suspend. Need to check)
# GSP (GPU System Processor) - this is a special chip which is present on NVIDIA video cards starting
# from Turing and above, which offloads GPU initialization and control tasks, which are usually
#performed on CPU. This should improve performance and reduce the load on the CPU.
#WARNING: I strongly suggest forcing the use of GSP firmware only on the most recent driver, as the first
#releases with its support may contain certain problems. Only starting from 530 you will have support
#for suspend and resume when using GSP firmware. This feature can also work badly on PRIME configurations,
#so please check dmesg logs for errors if you want to use this.

blacklist nouveau
alias nouveau off
#
# Nouveau must be blacklisted here as well beside from the initrd to avoid a
# delayed loading (for example on Optimus laptops where the Nvidia card is not
# driving the main display).

options nvidia_drm modeset=1

options nvidia_drm fbdev=1
# This options unlock new nvidia framebuffer
_EOF_
```

```bash
echo "nvidia-uvm" > /etc/modules-load.d/nvidia-uvm.conf
```

**Включаем интерфейсы питания от nvidia:**
```bash
systemctl enable nvidia-suspend.service nvidia-hibernate.service nvidia-resume.service
```

