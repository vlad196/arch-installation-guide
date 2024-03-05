В итоге понял, что с waybar легче всего использовать горячие клавиши

**Композитор**
paru -S hyprland-nvidia-git
**Что оттуда надо**
submap
cliphist
swaylock wlogout
**env**
https://wiki.archlinux.org/title/XDG_Desktop_Portal#xdg-desktop-portal-wlr_does_not_start_automatically_on_sway


**Демон обоев**
paru -S swww
лучше иметь заготовленные обои, не хочу городить скрипты

**Бар**
paru -S waybar-git
**модули**
Hyprland/Workspaces
Hyprland/Language
Keyboard State

clock & calendar
weather
pipewire
CPU
Taskbar
что надо расмотреть:
https://github.com/HarHarLinks/wireguard-rofi-waybar
Network
Memory


sudo usermod -a input vlad
paru -S aurutils checkupdates

**Менеджер приложений**
paru -S wofi

**Приложения окружения**
paru -S dolphin dolphin-plugins kdegraphics-thumbailers kitty-git gwenview

**Апплеты приложений**
paru -S network-manager-applet

**Оконный менеджер**
Paru -S sddm-git
sddm theme

**Демон уведомлений**
paru -S dunst-git


**nvim**

paru -S --needed tree-sitter-cli ripgrep gdu bottom npm
git clone --depth 1 https://github.com/AstroNvim/AstroNvim ~/.config/nvim
nvim

screenlock
paru -S waylock3

ЗВУК
alsa - звуковая архитектура
pipewire - фреймворк
pipewire-alsa piprewire-jack, pipewire-pulse - клиенты, которые заменяют сервера (waybar требует jack)
wireplumber - менеджер сеансов 
pavucontrol - микшер для pulseaudio (подключить к waybar)

Важно




Wireguard
nmcli connection import type wireguard file "$CONF_FILE" 
$CONF_FILE это файл конфигураций


`[wlr] [libseat] [libseat/backend/seatd.c:70] Could not connect to socket /run/seatd.sock: no such file or directory`
`LIBSEAT_BACKEND=logind`  в`/etc/environment`