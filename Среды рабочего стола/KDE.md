### Установка:
```bash
sudo -u vlad paru -Sy --needed plasma-meta kde-{graphics,system,utilities,multimedia,network,pim,sdk}-meta sddm power-profiles-daemon kde-cdemu-manager  xdg-desktop-portal-gtk
```

```bash
sudo -u vlad paru -S --asdep --needed {flatpak,plymouth}-kcm cracklib galera judy perl-dbd-mariadb python-{mysqlclient,libevdev,pyudev,yaml,gobject,lsp-server}  gtk3 gtk4 sshfs kplotting kleopatra languagetool unrar p7zip lzop lrzip arj dosfstools fatresize {exfat,nilfs}-utils {a,h}spell speech-dispatcher gst-libav kimageformats cryfs s-nail catdoc libappimage quota-tools freetds bluez-obex libwmf libopenraw webp-pixbuf-loader maliit-keyboard
```
>[!Note]
>Для роли phonon backend всегда выбираем VLC, т.к. на сегодня нормально [только он и поддерживается](https://community.kde.org/Distributions/Packaging_Recommendations#Non-Plasma_packages).
>При выборе tesdate нужно быть внимательней! Наш регион 96

**Включаем экранный менеджер:**
```bash
sudo systemctl enable sddm.service
```
Включаем power-profiles-daemon для powerdevil:
```bash
sudo systemctl enable power-profiles-daemon.service
```

#### sddm через wayland
```bash
mkdir -p /etc/sddm.conf.d && \
cp /usr/share/sddm/scripts/wayland-session /etc/sddm.conf.d/wayland-sesstion.conf
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
ExecStartPost=-/usr/bin/sleep 30
ExecStartPost=-/usr/bin/plymouth quit --retain-splash
_EOF_
```



# KDE настройка после перезагрузки
### Настройка Dolphin:
**Добавляем недостающие опциональные thumbnails для файлов:**
```bash
paru -S --asdeps --needed libheif taglib
```
**Добавляем дополнительные thumbnails для файлов:**
```bash
paru -S --needed libheif resvg raw-thumbnailer kde-thumbnailer-apk
```

### Автозагрузка:
**Делаем автозагрузку usbguard:**
```bash
cat << _EOF_ > $HOME/.config/autostart/usbguard-qt.desktop
[Desktop Entry]
Categories=System;
Comment=USBGuard-Qt
Exec=usbguard-qt
GenericName=USBGuard
Icon=usbguard-icon
Keywords=USB;USBGuard;Qt;
Name=USBGuard
TryExec=usbguard-qt
Type=Application
_EOF_
```

**Делаем автозагрузку yd-go:**
```bash
cat << _EOF_ > $HOME/.config/autostart/yd-go-applet.desktop
[Desktop Entry]
Categories=System;
Comment=Yandex-disk QT applet
Exec=yd-go
GenericName=Yandex Disk
Keywords=Yandex Disk;yd-go;Qt;
Name=yd-go-applet
TryExec=yd-go
Type=Application
_EOF_
```

### Включение темы breeze для приложений GTK в Flatpak.

**Удостоверяемся, что установлены порталы gtk:**
```bash
paru -S --asdep xdg-desktop-portal-gtk
```
**Для приложений GTK ставим тему breeze:**
```bash
flatpak install flathub org.gtk.Gtk3theme.Breeze
```
**Глобально внедряем папку с GTK настройками:**
```bash
mkdir -p ~/.local/share/flatpak/overrides && \
cat << _EOF_ > $HOME/.local/share/flatpak/overrides/global
[Context]
filesystems=xdg-config/gtk-3.0:ro;xdg-config/gtk-4.0:ro;
_EOF_
```
### Настройка ufw для kdeconnect:
Создание службы:
```bash
sudo bash -c 'cat << _EOF_ > /etc/ufw/applications.d/kde-connect
[Kde-connect]
title= Kde-connect
description=Kde-connect server
ports=1714
_EOF_'
```
Запуск службы:
```bash
sudo ufw allow Kde-connect
```

### Установить русскую раскладку:
```bash
cat << _EOF_ >> ~/.config/kxkbrc

[Layout]
DisplayNames=,
LayoutList=us,ru
Use=true
VariantList=,typo
_EOF_
```

### Добавляем сочетание клавиш смены языка как в gnome:
Увы, пока только через Gui настройки

### Включить Numlock в самой kde:
```bash
sed -i "/\[\$Version\]/{
n
n
a\\
\[Keyboard\]\\
NumLock=0 \n
}" $HOME/.config/kcminputrc
```

### Добавление аватара:
```bash
sudo curl -o /var/lib/AccountsService/icons/vlad https://besplatnye-programmy.com/uploads/posts/2021-04/1617720980_arch-linux.png
```
