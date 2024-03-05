### Добавление своего стека репозиториев:

beta-kde (Репозиторий с beta версией KDE. Когда нет теста, не имеет смысла ставить)
```bash
sed '/# Default repositories/i\
\# Beta-kde\
\[kde-unstable\]\
\Include = /etc/pacman.d/mirrorlist\
' -i /etc/pacman.conf
```

**Установка:**
```bash
sudo -u vlad paru -Sy --needed plasma-meta sddm kde-graphics-meta kde-system-meta kde-utilities-meta kde-multimedia-meta kde-network-meta ufw qt6-virtualkeyboard power-profiles-daemon phonon-qt6-vlc kmail
```

```bash
sudo -u vlad paru -Sy --asdeps cracklib galera judy perl-dbd-mariadb python-mysqlclient
```
>[!Note]
>Если ставиться plasma 5, необходим ещё один пакет для wayland plasma-wayland-session.
>Для роли phonon backend всегда выбираем VLC, т.к. на сегодня нормально [только он и поддерживается](https://community.kde.org/Distributions/Packaging_Recommendations#Non-Plasma_packages).

**Включаем экранный менеджер:**
```bash
systemctl enable sddm.service
```
Включаем power-profiles-daemon для powerdevil:
```bash
systemctl enable power-profiles-daemon.service
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

### SDDM:
**Создаём основной конфиг файл sddm(установим тему, numlock режим и т.д.):**
```bash
sudo bash -c 'cat << _EOF_ > /etc/sddm.conf.d/01_kde_settings.conf
[Autologin]
Relogin=false
Session=
User=

[General]
HaltCommand=/usr/bin/systemctl poweroff
RebootCommand=/usr/bin/systemctl reboot
Numlock=on


[Theme]
Current=breeze

[Users]
MaximumUid=60513
MinimumUid=1000
_EOF_'
```
**Указываем режим wayland и установленную виртуальную клавиатуру:**
```bash
mkdir /etc/sddm.conf.d/ && \
cat << _EOF_ >/etc/sddm.conf.d/10-wayland.conf
[General]
DisplayServer=wayland
CompositorCommand=kwin_wayland --drm --no-lockscreen --no-global-shortcuts --locale1 --inputmethod qtvirtualkeyboard
_EOF_
```
### Настройка Dolphin:
**Добавляем thumbnails для файлов:**
```bash
paru -S --needed kdegraphics-thumbnailers kimageformats5 libheif qt5-imageformats resvg kdesdk-thumbnailers  
ffmpegthumbs raw-thumbnailer taglib kde-thumbnailer-apk
```

### Автозагрузка:
**Делаем автозагрузку usbguard:**
```bash
sudo -u vlad bash -c 'cat << _EOF_ > /home/vlad/.config/autostart/usbguard-applet.desktop
[Desktop Entry]
Categories=System;
Type=Application
Name=Usbguard-applet
Icon=usbguard-icon
Comment=Usbguard QT applet
TryExec=usbguard-applet-qt
Exec=usbguard-applet-qt
_EOF_'
```

**Делаем автозагрузку yd-go:**
```bash
sudo -u vlad bash -c 'cat << _EOF_ > /home/vlad/.config/autostart/yd-go-applet.desktop
[Desktop Entry]
Categories=System;
Type=Application
Name=yd-go-applet
Comment=Yandex-disk QT applet
TryExec=yd-go
Exec=yd-go
_EOF_'
```
### KDE настройка после перезагрузки
**Настройка ufw для kdeconnect:**
```bash
sudo ufw allow 1714:1764/udp && \
sudo ufw allow 1714:1764/tcp
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
Включение gtk-breeze темы для flatpak
```bash
cat << _EOF_ >> ~/.local/share/flatpak/overrides/global
[Environment]
GTK_THEME=Breeze
ICON_THEME=Breeze
XCURSOR_THEME=Breeze
_EOF_
```

**Добавление аватара:**
```bash
sudo curl -o /var/lib/AccountsService/icons/vlad https://besplatnye-programmy.com/uploads/posts/2021-04/1617720980_arch-linux.png
```
