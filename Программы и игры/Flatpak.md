### Flatpak
**Установка:**
```bash
paru -S --needed flatpak
```
**Добавляем репозиторий flathub:**
```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

**Устанавливаем мои приложения:**
```bash
sudo flatpak install flathub md.obsidian.Obsidian \
org.telegram.desktop \
com.valvesoftware.Steam \
com.discordapp.Discord \
com.visualstudio.code \
com.microsoft.Edge
```

**Настройка Obsidian**
Добавляем wayland режим:
```bash
flatpak override --user --socket=wayland --env=OBSIDIAN_DISABLE_GPU=1 md.obsidian.Obsidian
```
>[!Note]
>Если нужны ещё разрешения: [flathub/md.obsidian.Obsidian (github.com)](https://github.com/flathub/md.obsidian.Obsidian)

**Настройка Discord**
Добавляем wayland режим:
1-ый вариант, который указан, но который не работает:
```bash
flatpak override --user --socket=wayland com.discordapp.Discord
```
>[!Note]
>Если нужны ещё разрешения: [flathub/com.discordapp.Discord (github.com)](https://github.com/flathub/com.discordapp.Discord)

2-ой вариант: 
1.Копируем ярлык и модифицируем его:
```bash
cp /var/lib/flatpak/app/com.discordapp.Discord/current/active/export/share/applications/com.discordapp.Discord.desktop ${HOME}/.local/share/applications/
```
2. Модифицируем его:
```bash
sed -i 's/\(Exec=.*\)/\1 --enable-features=UseOzonePlatform --ozone-platform=wayland/' ${HOME}/.local/share/applications/com.discordapp.Discord.desktop
```

**Настройка Microsoft Edge:**
Добавляем wayland режим:
```bash
cat << _EOF_ > "$HOME/.var/app/com.microsoft.Edge/config/edge-flags.conf"
--ozone-platform=wayland
--enable-features=UseOzonePlatform
_EOF_
```
**Настройка VSCode:** 
Добавляем  wayland:
1.Копируем ярлык и модифицируем его:
```bash
cp /var/lib/flatpak/app/com.discordapp.Discord/current/active/export/share/applications/com.visualstudio.code.desktop ${HOME}/.local/share/applications/
```
2. Модифицируем его:
```bash
sed -i 's|Exec=.* com.visualstudio.code|Exec=/usr/bin/flatpak run --branch=stable --arch=x86_64 --command=code --socket=wayland --file-forwarding com.visualstudio.code|g' ${HOME}/.local/share/applications/com.visualstudio.code.desktop && \
sed -i 's|@@ %F @@| --enable-features=UseOzonePlatform --ozone-platform=wayland @@ %F @@|g' ${HOME}/.local/share/applications/com.visualstudio.code.desktop
```
>[!Note]
>Источник: https://github.com/flathub/com.visualstudio.code/issues/471

Добавляем [gnome-keyring по-умолчанию](https://code.visualstudio.com/docs/editor/settings-sync#_troubleshooting-keychain-issues), чтобы обойти [проблему](https://github.com/microsoft/vscode/issues/189672):

```bash
sed -i 's|@@ %F @@| --password-store="basic" @@ %F @@|g' ${HOME}/.local/share/applications/com.visualstudio.code.desktop
```

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

