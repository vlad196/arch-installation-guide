### Дополнительные репозитории:
Gnome for fcgu (Репозиторий с самой быстрым анонсом гнома)
```bash
sed '/# Default repositories/i\
\# Gnome for fcgu\
\[fcgu\]\
\Include =\ /etc/pacman.d/fcgu-mirrorlist\
' -i /etc/pacman.conf
```

### Acct
**Установка:**
```bash
paru -S acct
```

>[!Info]
>За дополнительной информацией:  https://ru.linux-console.net/?p=2721&ysclid=lhjmit9prt310563175#gsc.tab=0

### sysstat
**Установка:**
```bash
paru -S sysstat
```
**Включение юнита sysstat:**
```bash
sudo systemctl enable --now sysstat
```

### AIDE
**Установка:**
```bash
paru -S aide
```
**Запуск:**
```bash
sudo aide -i
```

### Bash
>[!Info]
Oh-my-bash затирает весь bashrc, поэтому делаем бэкап, если там что-то важное

```bash
cp ./.bashrc ./bashrc.backup
```
**Устанавливаем oh my bash:**
```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/ohmybash/oh-my-bash/master/tools/install.sh)"
```
**Установка автодополнения команды:**
```bash
paru -S bash-completion-git
```

### SAMBA
**Устанавливаем:**
```bash
paru -S samba smbclient`
```
**Настройка конфига:**
```bash
sudo bash -c 'cat << _EOF_ > /etc/samba/smb.conf
[global]
security = user
workgroup = WORKGROUP
server string = Samba
#Тут пишем ваш ник вместо user
guest account = sambauser
map to guest = Bad User
auth methods = guest, sam_ignoredomain
create mask = 0664
directory mask = 0775
hide dot files = yes

[Сериалы]
comment = Public Folder
#Тут пишем путь до папки для шары
path = /mnt/sdb/Видео/Сериалы
browseable = Yes
guest ok = Yes
public = yes
writeable = Yes
read only = no
guest ok = yes
create mask = 0666
directory mask = 0775
_EOF_'
```
**Добавляем группу sambagroup:**
```bash
sudo groupadd -r sambagroup
```
**Добавляем польователя sambauser:**
```bash
sudo useradd sambauser
```
**Добавляем себя и sambauser в группу:**
```bash
sudo gpasswd sambagroup -a sambauser
```
```bash
sudo gpasswd sambagroup -a vlad
```
**Меняем папке владельца и группу:**
```bash
sudo chown vlad:sambagroup /mnt/sdb/Видео/Сериалы
```
**Меняем права группы в папке:**
```bash
sudo chmod -R 770 /mnt/sdb/Видео/Сериалы
```
**Добавляем пароль:**
```bash
sudo smbpasswd -a sambauser
```
**Запускаем сервис:**
```bash
sudo systemctl start smb.service
```
**Добавить правило в ufw:**
```bash
sudo ufw allow CIFS
```
**Убираем пользователя sambauser из sddm:**
```bash
sudo bash -c 'cat << _EOF_ > /etc/sddm.conf.d/hide_user.conf
[Users]
HideUsers=sambauser
_EOF_'
```
### Clevis:
**Устанавливаем необходимые пакеты:**
```bash
paru -S libpwquality clevis mkinitcpio-clevis-hook
```
**Проверяем распознаёт ли linux tpm модуль(Если при загрузке появляется ошибка tpm, то ничего страшного):**
```bash
test -c /dev/tpm0 && echo OK || echo FAIL
```
> [!INFO]
**Если нет, то проверяем включён ли TPM модуль в материнке:**

**Создаём копию заголовка (Её лучше сразу на какую-то флешку перекинуть):**
```bash
sudo cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /mnt/sdb/header-nvme0n1p2.img
```
%%
!!! НАДО ПОСМОТРЕТЬ, КАК ВОССТАНАВЛИВАТЬ ЭТИ ЗАГОЛОВКИ, НА БУДУЩЕЕ
%%
**Привязываем luks к clevis и внедряем в tpm ключ:**
```bash
sudo clevis luks bind -d /dev/nvme0n1p2 tpm2 '{"pcr_ids":"7"}'
```
**Проверяем всё ли записалось:**
```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```
**Добавить в mkinitcpio.conf clevis перед encrypt:**
```bash
sudo sed -i "/^HOOKS=/ s/^\(.*\)\(encrypt\)/\1clevis \2/" /etc/mkinitcpio.conf
```
**Обновляем ядра и подписываем их:**
```bash
sudo mkinitcpio -P ; sudo sbctl sign-all
```
## Если остались старые ключи от clevis (они остаются и после переустановеи системы), то можно их удалить:
**Проверяем лист привязанных паролей:**
```bash
sudo clevis luks list -d /dev/nvme0n1p2
```
**Чистим ключи tpm в модуле:**
```bash
sudo tpm2_clear
```
**Отвязываем ключи по очереди, где 1 в конце, это номер в списке:**
```bash
sudo clevis luks unbind -d /dev/nvme0n1p2 -s 1
```
#### Старый способ для работы windows.(Копирование на тот же раздел с linux)
> [!Если есть установленный Windows]
**Копирование windows boot:**
>```bash
>mount --mkdir /dev/sda1 /mnt/sda1 && \
>mkdir -p /efi/EFI/Boot && \
>mkdir -p /efi/EFI/Microsoft && \
>cp /mnt/sda1/EFI/Boot/bootx64.efi /efi/EFI/Boot/bootx64.efi && \
>cp -r /mnt/sda1/EFI/Microsoft /efi/EFI
>```

**Добавить опций для ядра в cmdline:**
```bash
echo "options lpj=3600036 raid=noautodetect rootfstype=f2fs" >> \
/etc/kernel/cmdline
```
>[!Info]
>**lpj=3600036** - уникальный параметр для каждой системы. Его значение автоматически определяется во время загрузки, что довольно трудоемко, поэтому лучше задать вручную. Определить ваше значение для lpj можно через следующую команду: `sudo dmesg | grep "lpj="`
>**raid=noautodetect** - отключает проверку на RAID во время загрузки. Если вы его используете - **НЕ** прописывайте данный параметр.
>**rootfstype=f2fs** - здесь указываем название файловой системы в которой у вас отформатирован корень.
>Доп. информация здесь: https://ventureo.codeberg.page/source/kernel-parameters.html
>**Эти опции по итогу на скорость никак не влияют и можно игнорировать**



### systemd-timesyncd
аналог nss-mdns
**Добавляем сервера из пула [NTP]([pool.ntp.org: the internet cluster of ntp servers (ntppool.org)](https://www.ntppool.org/ru/)):**
```bash
sudo mkdir /etc/systemd/timesyncd.conf.d/ && \
sudo bash -c 'cat << _EOF_ >> /etc/systemd/timesyncd.conf.d/local.conf
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.fr.pool.ntp.org
_EOF_'
```
**Включаем использование systemd-timesyncd для синхронизации времени:**
```bash
sudo timedatectl set-ntp true
```
**Включение юнита ntp:**
```bash
sudo systemctl enable --now systemd-timesyncd.service
```