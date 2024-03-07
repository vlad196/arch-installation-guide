beta-gnome (Репозиторий с beta версией GNOME)
```bash
sed '/# Default repositories/i\
\# Beta-gnome\
\[gnome-unstable\]\
\Include = /etc/pacman.d/mirrorlist\
' -i /etc/pacman.conf
```

**Установка:**

```bash
sudo -u vlad paru -Sy --needed baobab eog evince gdm gnome-backgrounds \
gnome-calculator gnome-calendar gnome-characters gnome-clocks \
gnome-color-manager gnome-console gnome-contacts gnome-control-center \
gnome-disk-utility gnome-font-viewer gnome-keyring gnome-logs gnome-menus \
gnome-photos gnome-music gnome-session gnome-settings-daemon gnome-shell \
gnome-shell-extensions gnome-software gnome-system-monitor gnome-text-editor \
gnome-tour gnome-user-docs gnome-user-share gnome-weather grilo-plugins gvfs \
gvfs-afc gvfs-goa gvfs-google gvfs-gphoto2 gvfs-mtp gvfs-nfs gvfs-smb malcontent \
nautilus orca rygel sushi totem tracker3-miners xdg-desktop-portal-gnome \
xdg-user-dirs-gtk yelp dconf-editor endeavour file-roller \
geary gnome-boxes gnome-devel-docs \
gnome-multi-writer gnome-notes gnome-tweaks \
seahorse sysprof gnome-shell-extension-gsconnect pipewire-pulse gnome-browser-connector
```
> [!INFO]
> Для gnome-terminal и gedit сделали замену и оно уже в основном пакете https://www.reddit.com/r/gnome/comments/rq04ha/what_to_expect_in_gnome_in_2022/

**Включаем оконный менеджер:**
```bash
systemctl enable gdm.service
```

>[!Note]
>С драйвера версии 545 nvidia научилась работать без nvidia_drm.modeset=1 используя simpledrm, но правила gnome ещё не поменяли. Поэтому их всё равно надо загрушить
```bash
ln -s /dev/null /etc/udev/rules.d/61-gdm.rules
```


# Gnome настройка после перезагрузки
### Включение темы Adwaita для приложений QT в Flatpak.

**Удостоверяемся, что установлены порталы kde:**
```bash
paru -S --asdep xdg-desktop-portal-kde
```
Для приложений GTK ставим тему breeze:
```bash
flatpak install flathub org.kde.KStyle.Adwaita \
org.kde.PlatformTheme.QGnomePlatform
```

### Установка gnome browser connector:
```bash
paru -S gnome-browser-connector
```
**Настройка ufw для gsconnect:**
```bash
sudo ufw allow 1714:1764/udp && \
sudo ufw allow 1714:1764/tcp
```

### Интеграция Usbguard с gnome:
>[!Note]
>Предполагается, что группа usbguard уже существует и пользователь уже там

**Добавляем права на исполнение утилиты usbguard группе usbguard:**

```bash
cat << _EOF_ > /etc/sudoers.d/usbguard-group
%usbguard ALL=(ALL) /usr/bin/usbguard *
_EOF_
```

**Создаём polkit для группы usbguard в гноме:**
```bash
sudo bash -c 'cat << _EOF_ >> /etc/polkit-1/rules.d/70-allow-usbguard.rules
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
        subject.isInGroup("usbguard")) {
            return polkit.Result.YES;
    }
});
_EOF_'
```

**Включаем защиту при экране блокировки в гноме:**
```bash
gsettings set org.gnome.desktop.privacy usb-protection true
```
>[!Info]
>Это не полная защита. Если хочеться полной блокировки всех usb, тогда необходим, например usbguard-notifier. Но мне хватает и защиты при выходе в экран блокировки и сам usbguard-notifier имеет какие-то проблемы в сборке из AUR
>За дополнительной информацией: https://wiki.archlinux.org/title/USBGuard#Usage

### Указываем таким программам, как телеграмм файловый менеджер gnome:
```bash
cat << _EOF_ >> ~/.config/environment.d/envvars.conf
QT_QPA_PLATFORMTHEME=gtk3
_EOF_
```

### Включаем 3д-ускорение в xwayland:
```bash
gsettings set org.gnome.mutter experimental-features '["kms-modifiers"]'
```
Взято отсюда: https://download.nvidia.com/XFree86/Linux-x86_64/545.29.06/README/xwayland.html

### Чтобы gkr-pam не просился раньше времени и не выдавал ошибку:
```bash
sudo bash -c 'sed -i "/^auth[[:space:]]*optional/s/\(pam_gnome_keyring.so\)\(.*\)$/\1 only_if=gdm\2/" /etc/pam.d/gdm-password && \
sed "/^session[[:space:]]*optional/s/auto_start/only_if=gdm/" -i /etc/pam.d/gdm-password'
```

Ошибка в Арч вики: https://wiki.archlinux.org/title/GNOME/Keyring#Unable_to_locate_daemon_control_file

Взято отсюда: https://askubuntu.com/questions/1144153/how-to-solve-gkr-pam-unable-to-locate-daemon-control-file

