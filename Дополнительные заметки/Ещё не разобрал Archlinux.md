# Далее команды, которые нужно перевести в строку

You will notice that by default the umask is 022 under /etc/profile, so we can just update that to be 027. So in that file, change this section:
https://elatov.github.io/2017/06/install-lynis-and-fix-some-suggestions/#default-umask-in-etcprofile-or-etcprofiledcustomsh-could-be-more-strict-eg-027-auth-9328
https://tunnelix.com/auditing-linux-operating-system-with-lynis/


###disable core dumps https://www.cyberciti.biz/faq/disable-core-dumps-in-linux-with-systemd-sysctl/
/etc/security/limits.conf
* hard core 0
* soft core 0

### Увеличение rounds для шифрования пароля
/etc/login.defs
SHA_CRYPT_MIN_ROUNDS 50000
SHA_CRYPT_MAX_ROUNDS 100000


###Убираем возможность компиляции для группы "остальные"
 #sudo chmod o-rx /usr/bin/gcc
 #sudo chmod o-rx /usr/bin/as
#!!!не работает paru


#### (3 физических диска)
#### ||└───nvme0n1:
#### ||    | └──nvme0n1p1 -- efi 2gb
#### ||    └────nvme0n1p2 -- LUKS decrypt
#### ||          LUKS:system 
#### ||          | └───systemvg-swap 16 gb
#### ||          └─────systemvg-root 935 gb
#### |└─────sda -- windows 
#### └──────sdb -- дополнительный диск с ссылками папок в него

### Проверить сайты:
Экранирование: https://seogift.ru/tools/ehkranirovanie-specsimvolov/ https://onlinelinuxtools.com/escape-shell-characters
Проверка скриптов: https://www.shellcheck.net/

### suspend hyprland
nvim .config/hypr/hyprland.conf
exec-once = swayidle -w timeout 1800  'hyprctl dispatch exec systemctl suspend' # suspend after 30 mins

### xeyes
paru -S xorg-xeyes  

### keyboart layout
nvim .config/hypr/hyprland.conf
```bash
input {
    kb_layout = us, ru
    kb_variant =
    kb_model =
    kb_options = grp:alt_shift_toggle     
    kb_rules =
    follow_mouse = 1

    touchpad {
        natural_scroll = no
    }
```

nvim .config/waybar/config.jsonc    
```bash
	"modules-right": ["custom/padd","custom/l_end","network","bluetooth","pulseaudio","pulseaudio#microphone","custom/updates",**"custom/keyboard-layout"**,"custom/r_end","custom/l_end","tray","custom/r_end","custom/l_end","custom/wallchange","custom/mode","custom/wbar","custom/cliphist","custom/power","custom/r_end","custom/padd"],
```

```bash
,

    "custom/keyboard-layout": {
        "return-type": "json",
        "exec": "hypr-kbd-layout", 
        "format": "{}"
    },

    "tray": {
        "icon-size": 16,
        "spacing": 5
    },

```

```bash
1|28|bottom|( cpu memory ) ( clock )|( wlr/workspaces hyprland/window )|( network bluetooth pulseaudio pulseaudio#microphone custom/updates **custom/keyboard-layout**) ( tray ) 
```

### vue-cli-service: Permission denied
```bash
chmod -R a+x ./node_modules 
```


IKEv2
Как создать сертификаты
https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/ikev2-howto.md#linux

Подключить через cli:
```bash
sudo nmcli c add type vpn ifname -- vpn-type strongswan connection.id <insert connection name> connection.autoconnect no vpn.data 'address = <insert vpn server address>, certificate = <full path to the extracted ikev2vpnca.cer>, encap = no, esp = aes128gcm16, ipcomp = no, method = key, proposal = yes, usercert = <full path to the extracted vpnclient.cer>, userkey = <full path to the extracted vpnclient.key>, virtual = yes'
```
https://github.com/hwdsl2/setup-ipsec-vpn/blob/master/docs/ikev2-howto.md#linux


Yandex-disk-indicator

```
paru -S yandex-disk-indicator
```

```
exec-once = yandex-disk-indicator
```


### Связываем рабочие папки с дополнительным диском:
**Добавить ссылки Загрузок, документов И Т.П. с дополнительного диска на основную папку:**
```bash
ln -s /mnt/sdb/Изображения /home/vlad/ && \
ln -s /mnt/sdb/Видео /home/vlad/ && \
ln -s /mnt/sdb/Загрузки /home/vlad/ && \
ln -s /mnt/sdb/Документы /home/vlad/ && \
ln -s /mnt/sdb/Музыка /home/vlad/
```
> [!INFO]
> Только если Вам действительно нужно, чтобы эти папки были на другом диске

paru -S --needed bluedevil breeze breeze-gtk breeze-plymouth discover drkonqi flatpak-kcm     kactivitymanagerd kde-cli-tools kde-gtk-config kdecoration kdeplasma-addons kgamma5 khotkeys kinfocenter kmenuedit kpipewire kscreen kscreenlocker ksshaskpass ksystemstats kwayland-integration kwin-git kwrited layer-shell-qt libkscreen libksysguard milou oxygen oxygen-sounds plasma-browser-integration plasma-desktop plasma-disks plasma-firewall plasma-integration plasma-nm plasma-pa plasma-sdk plasma-systemmonitor plasma-welcome plasma-workspace plasma-workspace-wallpapers plymouth-kcm polkit-kde-agent powerdevil sddm-kcm systemsettings xdg-desktop-portal-kde terminal

polkit-kde-agent не удалять


paru -Rns cliphist waybar-hyprland-git swww swayidle dunst swaylock-effects-git xdg-desktop-portal-hyprland-git hy  
prland-nvidia-git wlogout wofi network-manager-applet pavucontrol
paru -Rns clamtk-gnome  gnome-text-editor nautilus eog evince
