### Flatpak
**Установка:**
```bash
paru -S --needed flatpak
```
**Добавляем репозитории flathub:**
```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

**Устанавливаем мои приложения:**
```bash
sudo flatpak install flathub md.obsidian.Obsidian \
org.telegram.desktop \
com.valvesoftware.Steam \
com.discordapp.Discord \
com.visualstudio.code\
com.microsoft.Edge
```

**Для Obsidian добавляем переменную для wayland:**
```bash
flatpak override --user --socket=wayland --env=OBSIDIAN_DISABLE_GPU=1 md.obsidian.Obsidian
```
>[!Note]
>Если нужны ещё разрешения: [flathub/md.obsidian.Obsidian (github.com)](https://github.com/flathub/md.obsidian.Obsidian)

**Для Discord добавляем переменную для wayland:**
```bash
flatpak override --user --socket=wayland com.discordapp.Discord
```
>[!Note]
>Если нужны ещё разрешения: [flathub/com.discordapp.Discord (github.com)](https://github.com/flathub/com.discordapp.Discord)

**Для Edge добавляем переменную для wayland:**
```bash
cat << _EOF_ > "${HOME}/.var/app/com.microsoft.Edge/config/edge-flags.conf"
--ozone-platform=wayland
--enable-features=UseOzonePlatform
_EOF_
```
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
