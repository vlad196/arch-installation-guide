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
sudo -u vlad paru -Sy --needed plasma-meta sddm kde-graphics-meta kde-system-meta kde-utilities-meta kde-multimedia-meta kde-network-meta ufw qt6-virtualkeyboard power-profiles-daemon kmail kio5-extras kdoctools5 flatpak-kcm plymouth-kcm
```

```bash
sudo -u vlad paru -Sy cracklib galera judy perl-dbd-mariadb python-mysqlclient python-libevdev python-pyudev gtk3 sshfs kplotting python-gobject kdepim-addons kleopatra kdepim-addons languagetool python-lsp-server unrar p7zip lzop lrzip arj dosfstools exfat-utils fatresize nilfs-utils aspell hspell speech-dispatcher gst-libav kimageformats cryfs s-nail catdoc libappimage quota-tools  xdg-desktop-portal-gtk kdepim-addons kde-cdemu-manager
```
>[!Note]
>Если ставиться plasma 5, необходим ещё один пакет для wayland plasma-wayland-session.
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
**Добавляем thumbnails для файлов:**
```bash
paru -Sy --asdeps --needed ffmpegthumbs kde-cli-tools kdegraphics-thumbnailers kio-admin purpose libheif qt6-imageformats resvg kdesdk-thumbnailers raw-thumbnailer taglib kde-thumbnailer-apk

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
cat << _EOF_ > "$HOME/.local/share/flatpak/overrides/global"
[Context]
filesystems=xdg-config/gtk-3.0:ro;xdg-config/gtk-4.0:ro;
_EOF_
```
### Настройка ufw для kdeconnect:
```bash
sudo ufw allow 1714:1764/udp && \
sudo ufw allow 1714:1764/tcp
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
