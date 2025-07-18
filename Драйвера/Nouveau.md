#### nvidia-firmware
>[!Note]
>Для видеокарт Maxwell необходимо вытащить прошивку из проприетарного драйвера.
```bash
mkdir -p /tmp/nouveau && cd /tmp/nouveau && \
wget http://us.download.nvidia.com/XFree86/Linux-x86_64/340.108/NVIDIA-Linux-x86_64-340.108.run && \
wget https://raw.github.com/envytools/firmware/master/extract_firmware.py && \
sh NVIDIA-Linux-x86_64-340.108.run --extract-only && \
python3 extract_firmware.py && \
mkdir /lib/firmware/nouveau && \
cp -d nv* vuc-* /lib/firmware/nouveau/ && \
cd ~
```


##### **Mesa:**
 **Установка vulkan-nouveau:**
```bash
sudo -u vlad paru -S --needed lib32-vulkan-nouveau vulkan-nouveau
```
 **Установка Mesa's Vulkan layers:**
```bash
sudo -u vlad paru -S --asdep --needed vulkan-mesa-layers lib32-vulkan-mesa-layers
```
 **Установка Open-source VDPAU drivers:**
```bash
sudo -u vlad paru -S --needed mesa-vdpau lib32-mesa-vdpau
```

#### Power Managemenet:
Для видеокарт Maxwell нет автоматической поддержки питания, но есть Reclocking.

#### Ранняя загрузка драйвера:
>[!NOTE]
Драйвер загружается сам во время хука KMS, но можно форсировать более раннюю загрузку, добавив nouveau в модули. Видеодрайвер загружается чуть раньше и нет переключения видеопотока во время текстовой загрузки (Нет мерцания экрана).
Как настрою саму работу драйвера, проверю. Если будет смысл, то оставлю.

**Добавляем модуль драйвера nouveau:**
```bash
sed -e 's/\(MODULES=(\)/\1nouveau/' -i /etc/mkinitcpio.conf.d/mkinitcpio.conf
```

#### Аппаратное ускорение:
>[!NOTE]
Всё делится на 3 части. 1 Карты с видео движком, 2 карты без видео движка и 3, те что не поддерживаются:
Чтобы проверить поддерживается или нет, надо смотреть тут: https://nouveau.freedesktop.org/VideoAcceleration.html
У меня NV166 (TU106) и пока не поддерживает.
```bash
sudo -u vlad paru -Sy --needed nouveau-fw
```
>[!NOTE]
**UPDATE:** на данный момент ведётся разработка vulkan video от [Khronos]([Khronos Blog - The Khronos Group Inc](https://www.khronos.org/blog/an-introduction-to-vulkan-video)), которая должна привнести общий формат ускорения для всех видеокарт через vulkan. На данный момент вопрос с аппаратным ускорением в подвешенном состоянии.

**Дополнительная информация:**
>[!Note]
**xf86-video-nouveau** из АрчВики не нужен, тк:
*Разработчик Nouveau Илья Миркин объяснил, что ускорение GLAMOR работает на графических процессорах NVIDIA G80 (GeForce 8) и новее, в то время как Nouveau фокусируется на поддержке, начиная со времен Riva TNT2. Кроме того, xf86-video-nouveau спроектирован так, чтобы быть простым и хорошо работать, этот драйвер DDX можно использовать без драйвера OpenGL от Nouveau (в отличие от драйвера настройки режима, когда требуется ускорение GLAMOR), и ранее у GLAMOR были некоторые ошибки с Nouveau (но теперь, похоже, это исправлено).*
*Мейнтейнер Nouveau DRM Бен Скеггс (Ben Skeggs) также сказал, что он за использование xf86-video-modesetting на оборудовании серии GeForce 8 и новее. Он также не планирует поддерживать драйвер xf86-video-nouveau DDX, кроме исправления ошибок. Таким образом, новое оборудование и новые функции будут доступны только в драйвере DDX с установкой режима, если другие независимые разработчики не предпримут никаких шагов для улучшения xf86-video-nouveau.
