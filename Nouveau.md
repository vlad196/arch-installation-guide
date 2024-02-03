>[!Warning]
>Пока что мы хотим попробовать все новый функции. Разработчик NVK предоставляет эту возможность через пропатченный mesa в AUR. Вместо mesa пакета ставим vulkan-nouveau-git

Nouveau в очень активной разработке, поэтому чем более поздние пакеты, тем лучше его работа
В частности необходим пакет linux-git и последняя mesa
#### Пакеты:
**Добавление ключей для chaotic-aur:**
```bash
pacman-key --recv-key 3056513887B78AEB --keyserver keyserver.ubuntu.com && \
pacman-key --lsign-key 3056513887B78AEB && \
pacman -U 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-keyring.pkg.tar.zst' 'https://cdn-mirror.chaotic.cx/chaotic-aur/chaotic-mirrorlist.pkg.tar.zst'
```
**Добавление репозиториев chaotic-aur (Репозиторий с последними скомилированными бинарниками ) vulkan-nouveau-git:**
```bash
sed '/# Default repositories/i\
\[chaotic-aur]\
\Include = /etc/pacman.d/chaotic-mirrorlist
' -i /etc/pacman.conf
```
>[!Warning]
>Перед установкой лучше закомментировать mesa-git репозиторий
Добавление linux-git.preset с учётом своего mkinitcpio-nouveau.conf
##### Linux-git
Добавление linux-git.preset с учётом своего mkinitcpio.conf:
```bash
cat << _EOF_ > /etc/mkinitcpio.d/linux-git.preset
# mkinitcpio preset file for the 'linux-git' package

ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux-git"
ALL_microcode=(/boot/*-ucode.img)

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux-git.img"
default_uki="/efi/EFI/Linux/arch-linux-git.efi"
default_options="--cmdline /etc/kernel/cmdline"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-git-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-git-fallback.efi"
fallback_options="-S autodetect"
_EOF_
```
**Установка linux-git:**
```bash
paru -S linux-git linux-git-headers
```
##### **Nouveau и Mesa:**
 **Установка vulkan-nouveau-git:**
```bash
sudo -u vlad paru -Sy --needed lib32-vulkan-nouveau-git vulkan-nouveau-git xf86-video-nouveau
```
---
**В будущем будет:**
**Добавление репозиториев mesa-git (Репозиторий с последними скомилированными бинарниками mesa)**
```bash
sed '/# Default repositories/i\
\[mesa-git\]\
\SigLevel = Never\
\Server = https://pkgbuild.com/~lcarlier/\$repo\/\$arch\
' -i /etc/pacman.conf
```

**Установка mesa-git:**
```bash
sudo -u vlad paru -Sy mesa-git lib32-mesa-git xf86-video-nouveau
```
---
##### Power Managemenet:
Для видеокарт Ampere и Turing можно включить поддержку управления питанием.
**В параметры ядра добавляем:**
```bash
sed -i -e 's/$/ nouveau.config=NvGspRm=1/' /etc/kernel/cmdline
```

>[!Note]
**xf86-video-nouveau** из АрчВики не нужен, тк:
*Разработчик Nouveau Илья Миркин объяснил, что ускорение GLAMOR работает на графических процессорах NVIDIA G80 (GeForce 8) и новее, в то время как Nouveau фокусируется на поддержке, начиная со времен Riva TNT2. Кроме того, xf86-video-nouveau спроектирован так, чтобы быть простым и хорошо работать, этот драйвер DDX можно использовать без драйвера OpenGL от Nouveau (в отличие от драйвера настройки режима, когда требуется ускорение GLAMOR), и ранее у GLAMOR были некоторые ошибки с Nouveau (но теперь, похоже, это исправлено).*
*Мейнтейнер Nouveau DRM Бен Скеггс (Ben Skeggs) также сказал, что он за использование xf86-video-modesetting на оборудовании серии GeForce 8 и новее. Он также не планирует поддерживать драйвер xf86-video-nouveau DDX, кроме исправления ошибок. Таким образом, новое оборудование и новые функции будут доступны только в драйвере DDX с установкой режима, если другие независимые разработчики не предпримут никаких шагов для улучшения xf86-video-nouveau.

>[!Note]
Важные пакеты для сессии с nouveau. Они скачиваются в виде зависимостей с mesa, тем не менее полезно знать для будущего мониторинга:
>- mesa-git 
>- lib32-mesa-git
>- libdrm 
>- lib32-libdrm
>- vulkan-icd-loader
>- lib32-vulkan-icd-loader
> На момент января 2024 года все самые последние сборки лежат в основных репозиториях Extra и Multilib. 
> Есть git скрипты сборки в AUR и по идее они должны быть новее, но на данный момент они старее, чем в основном. Если ситуация поменяется, имеет смысл перейти на AUR скрипты.
#### Аппаратное ускорение:
>[!Info]
Всё делится на 3 части. 1 Карты с видео движком, 2 карты без видео движка и 3, те что не поддерживаются:
Чтобы проверить поддерживается или нет, надо смотреть тут: https://nouveau.freedesktop.org/VideoAcceleration.html
У меня NV166 (TU106) и пока не поддерживает.
```bash
sudo -u vlad paru -Sy --needed nouveau-fw
```
#### Ранняя загрузка драйвера:
>[!Info]
Драйвер загружается сам во время хука KMS, но можно форсировать более раннюю загрузку, добавив nouveau в модули. Видеодрайвер загружается чуть раньше и нет переключения видеопотока во время текстовой загрузки (Нет мерцания экрана).
Как настрою саму работу драйвера, проверю. Если будет смысл, то оставлю.
Без этого

**Добавляем модуль драйвера nouveau:**
```bash
sed -e 's/\(MODULES=(\)/\1nouveau/' -i /etc/mkinitcpio.conf.d/mkinitcpio.conf 
```

---
#### Power Managemenet:
Для видеокарт Ampere и Turing можно включить поддержку управления питанием.
**Добавляем в modprobe.d:**

```bash
cat << _EOF_ > /etc/modprobe.d/nouveau-power-management.conf
options nouveau config=NvGspRm=1
_EOF_
```