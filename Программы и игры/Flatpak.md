### Flatpak
**Установка:**
```bash
paru -S --needed flatpak
```
**Добавляем репозитории flathub и kdeapps:**
```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo && \
flatpak remote-add --if-not-exists  kdeapps https://distribute.kde.org/kdeapps.flatpakrepo
```
**Устанавливаем темы для gtk и kde:**
Удостоверяемся, что установлены порталы kde и gtk:
```bash
paru -S --asdep xdg-desktop-portal-gtk xdg-desktop-portal-kde
```
Для приложений gtk тему breeze:
```bash
flatpak install flathub org.gtk.Gtk3theme.Breeze
```
Глобально внедряем папку с gtk темами:
```bash
cat << _EOF_ > "~/.local/share/flatpak/overrides/global"
[Context]
filesystems=xdg-config/gtk-3.0:ro;xdg-config/gtk-4.0:ro;
_EOF_
```

Для приложений QT тему Adwaita:
```bash
flatpak install kdeapps org.kde.KStyle.Adwaita \
org.kde.PlatformTheme.QGnomePlatform
```

**Устанавливаем мои приложения:**
```bash
sudo flatpak install flathub md.obsidian.Obsidian \
org.telegram.desktop \
com.valvesoftware.Steam \
com.discordapp.Discord \
com.visualstudio.code
```

TODO: переделать в файл. Так только на сеанс
**Для Obsidian добавляем переменную для wayland:**
```bash
flatpak override --user --socket=wayland --env=OBSIDIAN_DISABLE_GPU=1 md.obsidian.Obsidian
```
>[!Note]
>Если нужны ещё разрешения: [flathub/md.obsidian.Obsidian (github.com)](https://github.com/flathub/md.obsidian.Obsidian)

**Для Discord добавляем переменную для wayland:**
```bash
cat << _EOF_ > "~/.local/share/flatpak/overrides/md.obsidian.Obsidian"
[Context]
sockets=wayland;fallback-x11;

[Environment]
OBSIDIAN_DISABLE_GPU=1
_EOF_
```
>[!Note]
>Если нужны ещё разрешения: [flathub/com.discordapp.Discord (github.com)](https://github.com/flathub/com.discordapp.Discord)

**Настройка VSCode:** 
Использование своей оболочки внутри vscode:
Добавляем в `File -> Preferences -> Settings -> Terminal > Integrated > Profiles`:
```bash
{
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.profiles.linux": {
    "bash": {
      "path": "host-spawn",
      "args": ["bash"]
    }
  }
}
```
>[!Note]
>Если нужны ещё разрешения: https://github.com/flathub/com.visualstudio.code
>Информация о flatpak внутри самого vscode: 
>==app/share/vscode/flatpak-warning.txt==

flatpak run  --socket=wayland com.visualstudio.code --disable-gpu --ozone-platform-hint=auto --enable-features=Way  
landWindowDecorations --disable-gpu-sandbox --ozone-platform=wayland